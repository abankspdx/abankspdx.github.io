---
title: "Python Codat Sync"
date: 2023-01-17T22:57:09-08:00
draft: false
---

## Python Scripting!

Recently I was tasked with writing a quick sync script to pull data from Codat (https://www.codat.io/). I haven't written the output yet (currently just JSON), but for the purposes of this post I thought my impl was good enough. I also omitted things like error notifications and remote logging (to be implemented later) - I kind of just wanted to get some thoughts out there with regard to structuring a script that syncs the contents of an external REST API.

### Tools

Almost none. For this script I just used Python3.10 and the `requests` library. Certainly you could get more complex, but I chose not to.

### Getting started

When going through the Codat API/swagger docs I found there were two "bases" that a RESTful entity could live under (ignoring connection-specific endpoints for now)

```
/companies/{company_id}/data/
/companies/{company_id}/data/financials/
```

As a normal data engineer, I wanted all of it, so I structured my code with some consts that included these bases to be formatted later. My code begins as:

```python
import requests
import json

CODAT_BASE = "https://api.codat.io"

COMPANY_ENDPOINT_BASE = "/companies/{}/data/"
COMPANY_ENDPOINTS = ["accounts", "billCreditNotes", "billPayments", "bills", "creditNotes", "customers", "info", "invoices", "items", "journalEntries", "journals", "paymentMethods", "payments", "purchaseOrders", "salesOrders", "suppliers", "taxRates", "trackingCategories"]
COMPANY_ENDPOINTS_TESTING = ["accounts", "bills", "invoices"]

FINANCIAL_ENDPOINT_BASE = "/companies/{}/data/financials/"
FINANCIALS_ENDPOINTS = ["balanceSheet", "profitAndLoss", "cashFlowStatement"]
```

This way, I could format each URL with its appropriate base and its appropriate endpoint without duplicating a bunch of work.

### Helpers

Before I move into the meat of the code, I'll include the two helper functions I wrote just to make life a little easier.

```python
def headers(auth_token: str) -> dict[str, str]:
    return {
        'Authorization': f"Basic {auth_token}",
        "Content-Type": "application/json",
        "Accept": "application/json"
    }

def format_endpoint(endpoint_base: str, endpoint: str, company_id: str) -> str:
    formatted_base = endpoint_base.format(company_id)
    return f"{CODAT_BASE}{formatted_base}{endpoint}"
```

These simply return the correct, formatted headers for each request + the full URL of the formatted endpoint, with the base + endpoint + company ID all included.


### Pagination

Each endpoint I intend to hit from Codat is paginated. As I said, I'm a normal data engineer, so I want _all the data_. Obviously this means I need to tackle pagination for each entity type. With python, easy!

I wrote two functions, an "outer" and an "inner" function, the inner being called repeatedly by the outer, and the outer being called by everything else. I did this to keep code clean and the function signatures for my outer functions simple.


```python
def get_paginated_result(endpoint: str, auth_token: str) -> list:
    shouldContinue = True
    result = []
    page = 0
    while shouldContinue:
        page += 1
        pageResult, sc = _get_paginated_result_with_page(endpoint, page, auth_token)
        result = result + pageResult
        shouldContinue = sc

    return result

def _get_paginated_result_with_page(endpoint: str, page: int, auth_token: str):
    params = {
        "page": page,
        "pageSize": 100
    }

    print("Getting endpoint: ", endpoint, " with params: ", json.dumps(params))

    response = requests.get(endpoint, headers=headers(auth_token), params=params)
    if response.status_code != 200:
        print(response.status_code)
        print(response.content)
        return [], False

    as_json = json.loads(response.content)
    to_return = as_json.get('results', [])

    if len(to_return) == 0:
        return to_return, False
    else:
        return to_return, True
```

In the first funciton, `get_paginated_result`, I simply take an endpoint and set up some variables (`shoudlContinue`, `result` and `page`). Then, while `shouldContinue` is true, pass those variables to the "inner" function, `_get_paginated_result_with_page`, which takes an endpoint and a page.

`_get_paginated_result_with_page` is where things are really interesting. First, I build the query params to attach to the endpoint so Codat knows the size of each page and which page I want. I hard coded page size to 100 for simplicity/smaller request sizes, but from their docs they support up to 5000. Then, I use the `requests` library to make a request with the appropriate headers and params. If the result isn't a 200, log it, and if it _is_ a 200, check to see if the resultset is empty. This is what happens if, as an example, there are 200 items and you're asking for page 2 with a page size of 200, ie, there are no more items to return. A 200 with an empty body signifies you're done pulling data for that entity, so return your resultset and a `shouldContinue` value of `False`. This way, the "outer" function knows to move on with a different endpoint.

### Tying it all together

The last piece is obviously calling the `get_paginated_result` function with each desired endpoint. That looks like this:

```python
def scrape_it_up(company_id: str, auth_token: str) -> dict:
    result = {}
    for endpoint in COMPANY_ENDPOINTS:
        full_url = format_endpoint(COMPANY_ENDPOINT_BASE, endpoint, company_id)
        res = get_paginated_result(full_url, auth_token)
        result[endpoint] = res

    for endpoint in FINANCIALS_ENDPOINTS:
        full_url = format_endpoint(FINANCIAL_ENDPOINT_BASE, endpoint, company_id)
        res = get_paginated_result(full_url)
        result[endpoint] = res

    return result
```

As you can see, I simply iterate over each endpoint in both of the aforementioned endpoint lists and attach their fully scraped resultsets to a dictionary whose key is the endpoint itself. This means the resultset will look like:

```json
{
    "bills": [...],
    "invoices": [...]
}
```

Which is very useful for downstream processing


### Conclusion

So now, the full code looks like this:

```python
import requests
import json

CODAT_BASE = "https://api.codat.io"

COMPANY_ENDPOINT_BASE = "/companies/{}/data/"
COMPANY_ENDPOINTS = ["accounts", "billCreditNotes", "billPayments", "bills", "creditNotes", "customers", "info", "invoices", "items", "journalEntries", "journals", "paymentMethods", "payments", "purchaseOrders", "salesOrders", "suppliers", "taxRates", "trackingCategories"]
COMPANY_ENDPOINTS_TESTING = ["accounts", "bills", "invoices"]

FINANCIAL_ENDPOINT_BASE = "/companies/{}/data/financials/"
FINANCIALS_ENDPOINTS = ["balanceSheet", "profitAndLoss", "cashFlowStatement"]

def headers(auth_token: str) -> dict[str, str]:
    return {
        'Authorization': f"Basic {auth_token}",
        "Content-Type": "application/json",
        "Accept": "application/json"
    }

def scrape_it_up(company_id: str, auth_token: str) -> dict:
    result = {}
    for endpoint in COMPANY_ENDPOINTS_TESTING:
        full_url = format_endpoint(COMPANY_ENDPOINT_BASE, endpoint, company_id)
        res = get_paginated_result(full_url, auth_token)
        result[endpoint] = res

    for endpoint in FINANCIALS_ENDPOINTS:
        full_url = format_endpoint(FINANCIAL_ENDPOINT_BASE, endpoint, company_id)
        res = get_paginated_result(full_url)
        result[endpoint] = res

    return result

def get_paginated_result(endpoint: str, auth_token: str) -> list:
    shouldContinue = True
    result = []
    page = 0
    while shouldContinue:
        page += 1
        pageResult, sc = _get_paginated_result_with_page(endpoint, page, auth_token)
        result = result + pageResult
        shouldContinue = sc

    return result

def _get_paginated_result_with_page(endpoint: str, page: int, auth_token: str):
    params = {
        "page": page,
        "pageSize": 100
    }

    print("Getting endpoint: ", endpoint, " with params: ", json.dumps(params))

    response = requests.get(endpoint, headers=headers(auth_token), params=params)
    if response.status_code != 200:
        print(response.status_code)
        print(response.content)
        return [], False

    as_json = json.loads(response.content)
    to_return = as_json.get('results', [])

    if len(to_return) == 0:
        return to_return, False
    else:
        return to_return, True

def format_endpoint(endpoint_base: str, endpoint: str, company_id: str) -> str:
    formatted_base = endpoint_base.format(company_id)
    return f"{CODAT_BASE}{formatted_base}{endpoint}"

demo_company_id = "12345"
demo_key = "11111"
result = scrape_it_up(demo_company_id, demo_key)
with open("./output.json", "a") as f:
    f.write(json.dumps(result))
```

And we're done! 70ish lines of Python gets us all the juicy Codat data we're looking for. Obviously there are improvements to be made if necessary. Things like logging, error handling, retries, but then also adding some functionality to parallelize retrieving data from each endpoint (which will also bring some complexity, having to manage open HTTP connections) to speed up ingest time.

The purpose of this code is to be usable by whatever runner is available. Fivetran as a custom connector, AWS SAM as a triggered function, Apache Airflow etc. This code specifically just needs the `requests` library and some working keys -- what it does with the data after it retrieves it is TBD!

Also tests. I think writing valuable tests for scripts like this is hard, because I have a general dislike for mocks, but then if you're not mocking your tests may also take a long time, but...tests are good and you should do them. Same for me.

Anyway, thanks for reading!