# UN-CNVc: the Unimaginatively-Named Copy Number Variant Caller

> We would like to thank the United Nations for serendipitously lending their name to our caller.

## Rationale
Unlike all of the CNV callers we know, `UN-CNVc` uses VCFs to call copy number variants. It is able to call CNVs on 1,500 samples using VCFs called using 20x depth WGS data in _20 minutes_ with basic mapping/reducing.

## Prerequisites
To run UN-CNVc, you will need :
* a recent (>v. 1.1) version of `bcftools`
* a recent (>v. 3.3) version of `R`
* the `mixtools` and `data.table` R libraries (`install.packages("mixtools")`)

## Usage

* First, generate per-sample, per-chromosome depths with `for i in {1..22}; do bcftools stats -s ... chr$i.vcf.gz; done`. Then extract relevant fields with `grep '^PSC' | cut -f3,10 > chr$i.avgdepth` This is going to be our baseline depth.
* Generate per-sample site-level depths with `bcftools view -r [region of interest] chr$i.vcf.gz  | bcftools query -f '%CHROM\t%POS\t[%DP\t]\n' > input.dp`. This is the main input of our script.
* Downsample depth to piecewise constant functions with `./call_sv.R input.dp chr$i.avgdepth out.sv`. (This step and the above can be run using our basic wrapper `searchsv.sh`)
* Call and genotype SVs:

```bash
./genotype_sv_new.R [input] [chromosome] [output_filename] [include_duplications]
```
* Options:
   * `[input]` is the file generated by `call_sv.R`.
   * `[chromosome]` is the chromosome number. Used only for the output.
   * `[output_filename]` is the basename of all output files.
   * `[include_duplications]` is an integer. 
      * `0` calls only deletions and calls duplications the same as no CNV. This generates a PLINK `.ped/.map` dataset.
      * `1` calls duplications as well as deletions. This generates a PLINK CNV dataset (`.cnv`).

## Output

* `[output_filename].stats`. A file with the following header, used to filter your CNVs:
   * `chr` 
   * `start`
   * `stop`
   * `hom` : number of homozygous deletions (depth=0)
   * `het` : number of heterozygous deletions (depth=0.5)
   * `normal` : number of samples with no variation
   * `dup` : number of samples with duplications (depth=k.0.5, with k>2)
   * `del_af` : frequency of deletion
   * `del_ratio` : ratio of deletions to duplications
   * `avg_del_p` : average confidence of the deletion calls
   * `del_hc` : number of high-confidence calls
   * `del_lc` : number of low-confidence calls (`confidence=5%` by default so low-confidence is really low confidence.)
   * `del_hc_total_ratio` : fraction of dels that are high-confidence 
   * `avg_dup_p` : average confidence of duplications
   * `dup_hc` : number of high-confidence duplication calls
   * `dup_lc`  : number of low-confidence dup calls
   * `dup_hc_total_ratio` : fraction of dups that are high-confidence
* `[output_filename].pdf` A diagnostics plot, used to visualise your results.
![example.png](example.png)
   * The top left panel represents the piecewise constant relative depth intervals. There will be a cluster around 1, representing the normal depth. Horizontal dashed intervals represent expected locations of CNV segments (het/hom del or dup). Vertical highlighted regions are regions called as variants by UN-CNVc.
   * Top right panel is a histogram of the observed segment depths.
   * Bottom left panel represents the statistics used by the caller to call variable regions.
* `[output_filename].ped/.map` PLINK dataset. Produced when `include_duplications=0`.
* `[output_filename].cnv` PLINK CNV dataset. Produced when `include_duplications=1`. Currently PLINK support for this format is in beta status.
