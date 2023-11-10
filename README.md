Goodreads
=========
```
SHELF='XXXXXXXXXXXXX'
curl https://www.goodreads.com/review/custom_widget/$SHELF's%20bookshelf:%20to-read?cover_position=left&cover_size=small&num_books=100&order=a&shelf=to-read&show_author=1&show_cover=0&show_rating=0&show_review=0&show_tags=0&show_title=1&sort=date_added&widget_bg_color=FFFFFF&widget_bg_transparent=&widget_border_width=1&widget_id=1699625535&widget_text_color=000000&widget_title_size=medium&widget_width=medium
```
Parse it with regex or beautiful soup to get the titles and authors. 

NLB
===
First, search the title/author to get the BRN. Remember to URL escape and to filter for format->code == BK for Books. Also for some reason, the author/title might not match what you want, so do another matching again.
```
export AUTHOR='Cormac%20McCarthy' TITLE='The%20Road'; curl -vH "X-Api-Key: $(lpass show nlb_dev_token  --password)" -H "X-App-Code: $(lpass show nlb_dev_token  --username)" "https://openweb.nlb.gov.sg/api/v1/Catalogue/SearchTitles?Author=${AUTHOR}&Title=${TITLE}" | jq '.'
```
A sample response is
```
{
  "setId": 527247742,
  "totalRecords": 7,
  "lastIrn": 3968361,
  "hasMoreRecords": false,
  "nextRecordsOffset": 0,
  "titles": [
    {
      "brn": 203431181,
      "irn": 267978936,
      "title": "On reading well : finding the good life through great books",
      "author": "Prior, Karen Swallow",
      "isbns": [
        "9781587433962 (hardcover ; : alkaline paper)",
        "1587433966 (hardcover ; : alkaline paper)"
      ],
      "publishDate": "2018",
      "format": {
        "code": "BK",
        "name": "Books"
      }
    },
    {
      "brn": 205518928,
      "irn": 362885772,
      "title": "The road",
      "author": "McCarthy, Cormac, 1933-",
      "isbns": [
        "9781509870639 (paperback)",
        "1509870636 (paperback)"
      ],
      "publishDate": "2019",
      "format": {
        "code": "BK",
        "name": "Books"
      }
    },
    {
      "brn": 205855177,
      "irn": 395292315,
      "title": "The road",
      "author": "McCarthy, Cormac, 1933-",
      "isbns": [
        "1035003791 (paperback)",
        "9781035003792 (paperback)"
      ],
      "publishDate": "2022",
      "format": {
        "code": "BK",
        "name": "Books"
      }
    },
    {
      "brn": 14424408,
      "irn": 1561314,
      "title": "The road",
      "publishDate": "2010",
      "format": {
        "code": "VM",
        "name": "Visual Materials"
      }
    },
    {
      "brn": 13107155,
      "irn": 2757374,
      "title": "The road",
      "author": "McCarthy, Cormac, 1933-",
      "isbns": [
        "0307455297 (paperback)",
        "9780307455291 (paperback)"
      ],
      "publishDate": "2008",
      "format": {
        "code": "BK",
        "name": "Books"
      }
    },
    {
      "brn": 13606556,
      "irn": 3190244,
      "title": "The road",
      "publishDate": "2010",
      "format": {
        "code": "VM",
        "name": "Visual Materials"
      }
    },
    {
      "brn": 12782246,
      "irn": 3968361,
      "title": "The road",
      "author": "McCarthy, Cormac, 1933-",
      "isbns": [
        "0307265439"
      ],
      "publishDate": "2006",
      "format": {
        "code": "BK",
        "name": "Books"
      }
    }
  ]
}
```

I recommend aggressively caching the BRNs of titles that I am interested in. BRNs are unlikely to change at all and I will need to query for like 100s of titles.
The cache will never get stale, at most titles might get added/removed from my toread, in which case I might need to cache more BRNs
Then, for each BRN, use the below API to search for availability:

```
export BRN='13107155';  curl -vH "X-Api-Key: $(lpass show nlb_dev_token  --password)" -H "X-App-Code: $(lpass show nlb_dev_token  --username)" "https://openweb.nlb.gov.sg/api/v1/Catalogue/GetAvailabilityInfo?BRN=${BRN}" | jq '.'
```

The goal is to show availability by library, to answer the question, 
What titles can I borrow from Jurong West library right now?

To answer that, for each BRN:

if the BRN is NOT STOCKED in my preferred libraries, cache that result and not revisit again for a "long" time (maybe 1 month)
if the BRN is ONLOAN, cache that result and not revisit again until the loan is about to expire
if the BRN is AVAILABLE, cache that result and revisit it again every day up to a LIMIT

so the AVAILABLE ones might generate the most queries (since refreshed daily)
but I also don't borrow more than 3 books at any one time, so for any given
preferred library, as long as I can present at least 6 books, it's good enough
So if a BRN is AVAILABLE but it's libarry already has 6 books, then SKIP.


Lastly, time the queries to go out at 2-3 am to relieve load on the servers.
