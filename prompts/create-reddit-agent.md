Create a **Ragbits-based agent** that runs **once per day** based on following steps
1. create directory structure and naming convention
2. reads posts from `r/artificial` from the last 24 hours,
   1. add reddit authentication needed things 
   2. add time filtering for the last 24 hours
   3. add error handling add loggin
3. converts the collected posts to Markdown,
4. saves the Markdown to a file.
5. create cron-friendly CLI entrypoint
