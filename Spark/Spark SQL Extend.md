## Extension points in Spark

>Version2.2 +
>
>Customize the spark session:
>
>- Optimizer
>
>- Parser
>- Analyzer
>- Physical Planning Strategy Rules

**Note**: This is just an exprimemtal API, so it could change across releases.



Pass in customized rules by such methods:

- injectParser – Adds Parser extensions rules

- injectResolutionRule – Adds analyzer rules (FixPoints)

- injectPostHocResolutionRule – Adds postHoc resolution rules in the analyzer (Once)

- injectCheckRule – Adds check rules in the analyzer during the analysis phase

- injectOptimizerRule – Adds Optimizer rules

- injectPlannerStrategy – Adds physical planning strategy rules
