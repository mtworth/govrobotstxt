# Robots.txt Scanner

Webscraping as a toolset has been supercharged by the advent of large language models. Many organizations have taken steps to restrict webscraping by LLM companies and other data providers through the robots.txt file (https://www.dataprovenance.org/Consent_in_Crisis.pdf). I wanted to understand how government agencies are responding to these scrapers and wrote a quick script to check for robots.txt files and which agents they govern. The script focused on government agency websites, but could be tooled for other URLs. 

Here is the dataset I'm using from data.gov 

```python
import pandas as pd
gov_websites = pd.read_csv("https://raw.githubusercontent.com/cisagov/dotgov-data/main/current-full.csv")
gov_websites['URL'] = 'http://' + gov_websites['Domain name']
```


Here is a function for checking URL for robots.txt and extracting the agents. 

```python
import urllib.robotparser
import urllib.request
import socket
from http.client import RemoteDisconnected

def check_robots_txt(url):
    rp = urllib.robotparser.RobotFileParser()
    robots_txt_url = f"{url}/robots.txt"

    try:
        # Set the timeout for the request
        with urllib.request.urlopen(robots_txt_url, timeout=10) as response:
            if response.status != 200:
                return (False, [])
        
        rp.set_url(robots_txt_url)
        rp.read()

        user_agents = set()

        for entry in rp.entries:
            for useragent in entry.useragents:
                user_agents.add(useragent)

        return (True, list(user_agents))

    except (urllib.error.URLError, socket.timeout, RemoteDisconnected) as e:
        print(f"Error accessing {robots_txt_url}: {e}")
        return (False, [])

#implement function on gov websites 
gov_websites[['HasRobotsTxt', 'UserAgents']] = gov_websites['URL'].apply(lambda url: pd.Series(check_robots_txt(url)))

```


Here is a function for doing this process with multiprocessing in case you're impatient like me :). 100 URLs takes about 11 seconds. 

```python
from concurrent.futures import ThreadPoolExecutor, as_completed

def process_urls_multithreaded(df):
    urls = df['URL'].tolist()
    results = [None] * len(urls)

    def worker(url, index):
        results[index] = check_robots_txt(url)

    # Create a ThreadPoolExecutor
    with ThreadPoolExecutor(max_workers=20) as executor:
        # Submit all tasks to the executor
        futures = [executor.submit(worker, url, i) for i, url in enumerate(urls)]
        
        # Ensure all futures are completed
        for future in as_completed(futures):
            try:
                # No need to do anything here; results are collected in worker
                pass
            except Exception as e:
                print(f"Exception occurred: {e}")

    # Convert results to DataFrame and update the original DataFrame
    results_df = pd.DataFrame(results, columns=['HasRobotsTxt', 'UserAgents'])
    df[['HasRobotsTxt', 'UserAgents']] = results_df
```
