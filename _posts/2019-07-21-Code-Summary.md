---
layout: post
title: Python Code, Summary
subtitle: Commented code of all coding-exercises
tags: [digital humanities, beautiful soup, python, JSON, CSV]
comments: true
---

Code for extracting all the relevant articles of 'The Dispatch' and writing them into an TSV-file

``` python
import os
import csv
from bs4 import BeautifulSoup

source_folder = os.path.join(os.getcwd(), 'data') # expects The Dispatch xml-files in a subfolder named 'data'
target_folder = os.path.join(os.getcwd(), 'data_modified') # output subfolder
os.makedirs(target_folder, exist_ok=True) # caution: _exist_ok = True_ still raises FileExistsError *if* target path exists and isn't a directory
dptch_dict = []

for file in os.listdir(source_folder):
    with open(os.path.join(source_folder, file), 'r', encoding='latin-1') as xml_file: #, 'utf-8' doesn't work
        dptch_soup = BeautifulSoup(xml_file.read(), 'lxml-xml') # dptch_soup is a _BeautifulSoup object_, which represents the document as a whole
    dptch_date = dptch_soup.find('date', value=True)['value'] # finds the first date tags with the attribute 'value'; assigns the value from the key 'value', which is the date in string-format
    for c, div3_tag in enumerate(dptch_soup.find_all('div3', type=True), 1): # for-loop over a list of all the div3-tags that have a 'type' attribute; counter 'c' starts at 1
        if div3_tag.head is not None:
            if not div3_tag.head.contents: # 'not tag.head.contents' is true only for empty head-tags
                div3_header = 'NO HEADER'
            else:
                div3_header  = div3_tag.head.get_text()
        else:
            headEntry = 'NA'      
        dtmp = {
        'id': dptch_date + '_' + div3_tag['type'].replace(' ', '') + '_' + str(c),
        'date': dptch_date,
        'type': div3_tag['type'],
        'header': div3_header,
        'text': ' '.join([text for text in div3_tag.stripped_strings]) # using list comprehensions to make a list of strings void of any extra whitespace, and join them with '' (empty string) as seperator
        } # temporary dictionary
        dptch_dict.append(dtmp) # list of dictionaries

with open(os.path.join(target_folder, 'dispatch_test.tsv'), 'w') as output_file:
    keys = ['id', 'date', 'type', 'header', 'text'] # list of the column names
    dict_writer = csv.DictWriter(output_file, fieldnames=keys, delimiter='\t')
    dict_writer.writeheader() # write the header
    dict_writer.writerows(dptch_dict)

print('Finished!')
```

Code for collecting the frequency of places and their longitude and latitude

``` python
import os 
import csv
import requests
from bs4 import BeautifulSoup

def get_geodata(tgn): # returns a dict {name, lat, lon}, given by the tgn-key; else returns 'NA' ecept for tgn
    #gd_dict = {}
    req = requests.get(f'http://vocab.getty.edu/tgn/{tgn}-place') # a _Requests_ object called 'req'
    print('Requesting ', req.url)
    place_soup = BeautifulSoup(req.content, 'lxml') # a _BeautifulSoup_ object called 'place_soup'
    if place_soup.find('p', class_='caption') is not None: # _None_ is the sole singleton object of NoneType; thus if <p class="caption"> exists, the if-clause is valid
        place_name = 'NA'
        place_lat = 'NA'
        place_lon = 'NA'
    else:
        place_name = place_soup.find('title').get_text(strip=True)
        crdtmp = place_soup.find('table', style=True).find_all('td')[3].get_text(strip=True) # ugly, but it works
        try: # intercept strings, e.g. 'http://opendatacommons.org/licenses/by/1.0/'
            place_lat = float(crdtmp)
        except ValueError:
            place_lat = 'NA'
        crdtmp = place_soup.find('table', style=True).find_all('td')[5].get_text(strip=True)
        try: # intercept strings, e.g. 'http://opendatacommons.org/licenses/by/1.0/'
            place_lon = float(crdtmp)
        except ValueError:
            place_lon = 'NA'
    gd_dict = {
        'name' : place_name, 
        'lat' : place_lat, 
        'lon' : place_lon
        }
    return gd_dict

# ===== main =====

source_folder = os.path.join(os.getcwd(), 'data') # expects The Dispatch xml-files in a subfolder named 'data'
target_folder = os.path.join(os.getcwd(), 'data_modified') # output subfolder
os.makedirs(target_folder, exist_ok = True) # caution: _exist_ok = True_ still raises FileExistsError *if* target path exists and isn't a directory

place_freq = {} 
place_geodat = {}

for file in os.listdir(source_folder):
    with open(os.path.join(source_folder, file), 'r', encoding='latin-1') as xml_file: #, 'utf-8' doesn't work
        dptch_soup = BeautifulSoup(xml_file.read(), 'lxml-xml') # dptch_soup is a _BeautifulSoup object_, which represents the document as a whole
    dptch_date = dptch_soup.find('date', value=True)['value'] # finds the first date tags with the attribute 'value'; assigns the value from the key 'value', which is the date in string-format
    for div3_tag in dptch_soup.find_all('div3', type=True): # for-loop over a list of all the div3-tags that have a 'type' attribute
        if div3_tag['type'] == 'ad-blank':
            continue # skips the rest of the code inside the for-loop for the current iteration only *if* 'type' is 'ad-blank'
        for placename_tag in div3_tag.find_all('placeName', key=True): # for-loop over a list of all placeName-tags of the *current* div3-tag
            tgn_keys = placename_tag['key'].split(';') # if there is no delimiter, split() returns the orignial string; multiple tgn-keys are delimited by ';'
            for key in tgn_keys:
                tmpkey = [d for d in key if d.isdigit()] # using list comprehensions and str.isdigit() to extract the numeric tgn-key
                tgn_id = ''.join(tmpkey) # separator is '' (empty string); 'tng_id' is now set to the numeric tgn-key
                if tgn_id in place_freq: # updating the word frequency dictionary
                    place_geodat[tgn_id]['date'].append(dptch_date)
                    place_geodat[tgn_id]['freq'] += 1
                else: # create new dictionary-entries
                    place_geodat[tgn_id] = get_geodata(tgn_id) # creates a nested-dict; key is the tgn-key
                    if place_geodat[tgn_id]['name'] == 'NA': # maybe easier with update()?
                        place_geodat[tgn_id]['name'] = placename_tag.get_text(strip=True) # if the placename has no valid tgn-key (i.e. there is no website), then the placename is scraped from the text
                    place_geodat[tgn_id]['date'] = [dptch_date]
                    place_geodat[tgn_id]['freq'] = 1 

with open(os.path.join(target_folder, 'dispatch_geodata_freq.csv'), 'w') as csv_file:
    csv_writer = csv.writer(csv_file, delimiter = '\t', quoting = csv.QUOTE_NONE)
    csv_writer.writerow(['placename', 'time_parameter', 'latitude', 'longitude', 'frequency']) # write the header, i.e. column names
    for v in place_geodat.values(): # loops through the values of the nested-dict
        for d in v['date']: 
             csv_writer.writerow([v['name'], d, v['lat'], v['lon'], v['freq']]) # writes a row for every date 'd' with "name - date - lat - lon - freq"

print('Finished!')
```