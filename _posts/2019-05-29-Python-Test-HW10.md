---
layout: post
title: Python Test, Homework 10
subtitle: Only half way through
tags: [homework, digital humanities, beautiful soup, python, JSON, CSV]
comments: true
---

I've tried to follow the PEP 8 style guide for Python

``` python
import os 
import json
import csv
from bs4 import BeautifulSoup

entries_list = []
word_freq_dict = {}

source_folder = os.path.join(os.getcwd(), 'data')
list_of_files = os.listdir(source_folder)
target_folder = os.path.join(os.getcwd(), 'data_modified')

os.makedirs(target_folder, exist_ok = True)

for file_name in list_of_files:
    with open(os.path.join(source_folder, file_name), 'r', encoding = 'latin-1') as fr: #, 'utf-8' doesn't work
        data_orig = BeautifulSoup(fr.read(), 'lxml-xml')
    issue_date = data_orig.find('date', value = True) # finds the first date tags with the attribute 'value'
    place_name = data_orig.find_all('placeName', key = True)
    for (i, tag) in enumerate(place_name[1:]):
        keytmp = [d for d in tag['key'] if d.isdigit()] # using list comprehensions and str.isdigit() to extract the numeric tgn-key
        tag_id = ''.join(keytmp) # separator is '' (empty string); 'tag_id' is now set to the numeric tgn-key
        if tag_id in word_freq_dict: # updating the word frequency dictionary
            word_freq_dict[tag_id] += 1
        else:
            word_freq_dict[tag_id] = 1
        dtmp = {
        'id': tag_id, 
        'reg': tag['reg'], # 'tag' is the current tag; 'place_name' is the whole list
        'text': tag.get_text(),
        'date': issue_date['value']
        } # temporary dictionary
        entries_list.append(dtmp) # list of dictionaries
with open(os.path.join(target_folder, 'dispatch_locations.JSON'), 'w', encoding = 'utf-8') as jfw: # JSON file write (jfw)
    json.dump(entries_list, jfw) # json.dump != json.dumps; supports file-like object 
with open(os.path.join(target_folder, 'word_freq.csv'), 'w') as csv_file:
    wf_writer = csv.writer(csv_file, delimiter = '\t', quoting = csv.QUOTE_NONE)
    for key, value in word_freq_dict.items():
        wf_writer.writerow([key, value])
print('Finished!')
```

_Output:_

![JSON file content](/img/2019-05-29-Test-JSON.png)
![CSV file content](/img/2019-05-29-Test-CSV.png)
