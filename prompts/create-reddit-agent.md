Create a **Ragbits-based agent** that runs **once per day** based on following steps
1. create directory structure and naming convention
2. dont use REDDIT CLIENT ID and REDDIT CLIENT_SECRET instead use only read endpoint which ended ".json" 
3. reads posts from `r/artificial` from the last 24 hours,
   1. add reddit authentication needed things 
   2. add time filtering for the last 24 hours
   3. add error handling add loggin
4. converts the collected posts to Markdown,
5. saves the Markdown to a file.
6. create cron-friendly CLI entrypoint
