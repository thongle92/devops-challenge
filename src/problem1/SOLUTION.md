Provide your CLI command here: `jq -r 'select(.symbol == "TSLA") | "https://example.com/api/\(.order_id)"' ./transaction-log.txt | tee ./output.txt | xargs -I {} curl -s -w "\nURL: %{url_effective} | Status: %{http_code}\n\n" "{}"`.  
The output.txt file format:
- Response Body
- URL + status code  
Example:
```
<!doctype html><html lang="en"><head><title>Example Domain</title><meta name="viewport" content="width=device-width, initial-scale=1"><style>body{background:#eee;width:60vw;margin:15vh auto;font-family:system-ui,sans-serif}h1{font-size:1.5em}div{opacity:0.8}a:link,a:visited{color:#348}</style></head><body><div><h1>Example Domain</h1><p>This domain is for use in documentation examples without needing permission. Avoid use in operations.</p><p><a href="https://iana.org/domains/example">Learn more</a></p></div></body></html>

URL: https://example.com/api/12346 | Status: 404
```
