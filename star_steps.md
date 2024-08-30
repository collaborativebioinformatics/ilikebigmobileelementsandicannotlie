# toy data steps as described in the step-by-step tutorial

## setup
```
mkdir test
```

1. set the starfish dir
```
STARFISHDIR=$(dirname -- $(command -v -- starfish))/../
```

2. generate text files of sample\tfile_path
```
realpath test/assembly/* | perl -pe 's/^(.+?([^\/]+?).fasta)$/\2\t\1/' > ome2assembly.txt

realpath test/gff3/* | perl -pe 's/^(.+?([^\/]+?).final.gff3)$/\2\t\1/' > ome2gff.txt
```

3. useful for consolidating de novo predicted genes (unclear if needed)
```
cat test/gff3/*.gff3 > macpha6.gff3
```

4. blastn database construction
```
cut -f2 ome2assembly.txt | xargs cat > blastdb/macpha6.assemblies.fna
makeblastdb \
	-in blastdb/macpha6.assemblies.fna \
	-out blastdb/macpha6.assemblies \
	-parse_seqids \
	-dbtype nucl
```

5. calculate %GC content (useful for viz later?)
```
$STARFISHDIR/aux/seq-gc.sh \
	-Nbw 1000 \
	blastdb/macpha6.assemblies.fna \
	> macpha6.assemblies.gcContent_w1000.bed
```

6. parse the provided eggnog mapper annotations NOTE only works for old emapper data as with the toy data
```
cut -f1,12  test/ann/*emapper.annotations \
	| grep -v  '#' \
	| grep -v -P '\t-' \
	| perl -pe 's/\t/\tEMAP\t/' \
	| grep -vP '\tNA' \
	> test/ann/macph6.gene2emap.txt
```

7. retrieve the narrowest eggnog ortholog group per sequence and convert to mcl format
```
cut -f1,10 test/ann/*emapper.annotations \
	| grep -v '#' \
	| perl -pe 's/^([^\s]+?)\t([^\|]+).+$/\1\t\2/' \
	> test/ann/macph6.gene2og.txt
	
$STARFISHDIR/aux/geneOG2mclFormat.pl -i test/ann/macph6.gene2og.txt -o test/ann/
```

## gene finder (module 01)
```
mkdir geneFinder
```

1. de novo annotate tyrs with the provided YR HMM and amino acid queries. NOTE there are some comments about annotating with different genes (-i is the predicted gene featureID prefix)
```
starfish annotate \
	-T 2 \
	-x macpha6_tyr \
	-a ome2assembly.txt \
	-g ome2gff.txt \
	-p $STARFISHDIR/db/YRsuperfams.p1-512.hmm \
	-P $STARFISHDIR/db/YRsuperfamRefs.faa \
	-i tyr \
	-o geneFinder/
```

2. collapse newly predicted genes with existing concat GFF (macpha6.gff3 above)
```
starfish consolidate \
	-o ./ \
	-g macpha6.gff3 \
	-G geneFinder/macpha6_tyr.filt_intersect.gff
	
realpath macpha6_tyr.filt_intersect.consolidated.gff | perl -pe 's/^/macpha6\t/' > ome2consolidatedGFF.txt
```

3. organize tyrs into mutually exclusive neighborhoods separated by at least 10kb
```
starfish sketch \
	-m 10000 \
	-q geneFinder/macpha6_tyr.filt_intersect.ids \
	-g ome2consolidatedGFF.txt \
	-i s \
	-x macpha6 \
	-o geneFinder/	
```

4. get captain candidates coordinates
```
grep -P '\ttyr\t' geneFinder/macpha6.bed > geneFinder/macpha6.tyr.bed 
```

## element finder (module 02)
```
mkdir elementFinder
```

1. using the coordinates of the predicted tyrs as starting points for blast-based search
```
starfish insert \
	-T 2 \
	-a ome2assembly.txt \
	-d blastdb/macpha6.assemblies \
	-b geneFinder/macpha6.tyr.bed \
	-i tyr \
	-x macpha6 \
	-o elementFinder/
```

2. candidate tyrs that do NOT already have a predicted boundary will be searched with the new parameters
```
starfish insert \
	-T 2 \
	-a ome2assembly.txt \
	-d blastdb/macpha6.assemblies \
	-b elementFinder/macpha6.insert.bed \
	-i tyr \
	-x macpha6_round2 \
	-o elementFinder/ \
	--pid 80 \
	--hsp 750
```

3. earch for flanking direct repeats (DRs) and terminal inverted repeats (TIRs) around each predicted boundary associated with the captain tyrs
```
starfish flank \
	-a ome2assembly.txt \
	-b elementFinder/macpha6.insert.bed \
	-x macpha6 \
	-o elementFinder/
```

4. consolidate insert and flank
```
starfish summarize \
	-a ome2assembly.txt \
	-b elementFinder/macpha6.flank.bed \
	-x macpha6 \
	-o elementFinder/ \
	-S elementFinder/macpha6.insert.stats \
	-f elementFinder/macpha6.flank.singleDR.stats \
	-g ome2consolidatedGFF.txt \
	-A test/ann/macph6.gene2emap.txt \
	-t geneFinder/macpha6_tyr.filt_intersect.ids 
```

4a. optional apply colors
```
awk '{
    if ($4 ~ /DR/) print $0 "\t255,255,0";        # Yellow for 'DR' in column 4
    else if ($4 ~ /TIR/) print $0 "\t255,165,0";  # Orange for 'TIR' in column 4
    else if ($5 ~ /tyr|cap/) print $0 "\t255,0,0"; # Red for 'tyr' or 'cap' in column 5
    else if ($5 != ".") print $0 "\t128,0,128";   # Purple if column 5 is not '.'
    else print $0 "\t169,169,169";                # Dark gray otherwise
}' elementFinder/macpha6.elements.bed > elementFinder/macpha6.elements.color.bed
```

4b. Assign all "Starships" to a family by searching the candidate captain sequences against the reference captain HMM profile database and identifying the best profile hit to each sequence
```
hmmsearch --noali --notextw -E 0.001 --max --cpu 12 --tblout elementFinder/macpha6_tyr_vs_YRsuperfams.out $STARFISHDIR/db/YRsuperfams.p1-512.hmm geneFinder/macpha6_tyr.filt_intersect.fas
perl -p -e 's/ +/\t/g' elementFinder/macpha6_tyr_vs_YRsuperfams.out | cut -f1,3,5 | grep -v '#' | sort -k3,3g | awk '!x[$1]++' > elementFinder/macpha6_tyr_vs_YRsuperfams_besthits.txt
```

4c. replace all captain IDs with starship IDs to simplify downstream parsing
```
grep -P '\tcap\t' elementFinder/macpha6.elements.bed | cut -f4,7 > elementFinder/macpha6.cap2ship.txt
$STARFISHDIR/aux/searchReplace.pl --strict -i elementFinder/macpha6_tyr_vs_YRsuperfams_besthits.txt -r elementFinder/macpha6.cap2ship.txt > elementFinder/macpha6_elements_vs_YRsuperfams_besthits.txt
```

4d. group all Starships into naves using mmseqs2 easy-cluster on the captain sequences with a very permissive 50% percent ID/ 25% coverage threshold:
```
mmseqs easy-cluster geneFinder/macpha6_tyr.filt_intersect.fas elementFinder/macpha6_tyr elementFinder/ --threads 2 --min-seq-id 0.5 -c 0.25 --alignment-mode 3 --cov-mode 0 --cluster-reassign
$STARFISHDIR/aux/mmseqs2mclFormat.pl -i elementFinder/macpha6_tyr_cluster.tsv -g navis -o elementFinder/
```

5. use sourmash and mcl to group all elements into haplotypes based on pairwise k-mer similarities of element nucleotide sequences:
```
starfish sim -m element -t nucl -b elementFinder/macpha6.elements.bed -x macpha6 -o elementFinder/ -a ome2assembly.txt
starfish group -m mcl -s elementFinder/macpha6.element.nucl.sim -i hap -o elementFinder/ -t 0.05
```
6. replace captain IDs with starship IDs in the naves file:
```
$STARFISHDIR/aux/searchReplace.pl -i elementFinder/macpha6_tyr_cluster.mcl -r elementFinder/macpha6.cap2ship.txt > elementFinder/macpha6.element_cluster.mcl
```
7. merge navis with haplotype to create a navis-haplotype label for each Starship:
```
$STARFISHDIR/aux/mergeGroupfiles.pl -t elementFinder/macpha6.element_cluster.mcl -q elementFinder/macpha6.element.nucl.I1.5.mcl > elementFinder/macpha6.element.navis-hap.mcl
```
8. convert mcl to gene2og format to simplify downstream parsing:
```
awk '{ for (i = 2; i <= NF; i++) print $i"\t"$1 }' elementFinder/macpha6.element.navis-hap.mcl > elementFinder/macpha6.element.navis-hap.txt
```
9. now add the family and navis-haplotype into to the element.feat file to consolidate metadata:
```
join -t$'\t' -1 1 -2 2 <(sort -t$'\t' -k1,1 elementFinder/macpha6.element.navis-hap.txt | grep -P '_e|_s') <(sort -t$'\t' -k2,2 elementFinder/macpha6.elements.feat) | awk -F'\t' '{print}' > elementFinder/macpha6.elements.temp.feat
echo -e "#elementID\tfamilyID\tnavisHapID\tcontigID\tcaptainID\telementBegin\telementEnd\telementLength\tstrand\tboundaryType\temptySiteID\temptyContig\temptyBegin\temptyEnd\temptySeq\tupDR\tdownDR\tDRedit\tupTIR\tdownTIR\tTIRedit\tnestedInside\tcontainNested" > elementFinder/macpha6.elements.ann.feat
join -t$'\t' -1 1 -2 1 <(sort -t$'\t' -k1,1 elementFinder/macpha6_elements_vs_YRsuperfams_besthits.txt | grep -P '_e|_s' | cut -f1,2) <(sort -t$'\t' -k1,1 elementFinder/macpha6.elements.temp.feat) | awk -F'\t' '{print}' >> elementFinder/macpha6.elements.ann.feat
```
