# Lab 6: Add Machine Learning

## Task 2: Preparation

```sh
export APIKEY=AIzaSyBIANiNuTT3jtT5nzzGXW8gZVLyI_sEQmg
export BUCKET=qwiklabs-gcp-c742fc67df02460a
export DEVSHELL_PROJECT_ID=qwiklabs-gcp-c742fc67df02460a
echo $DEVSHELL_PROJECT_ID, $BUCKET, $APIKEY
```

```sh
cd
cp -r /training/training-data-analyst .
ls

cd ~/training-data-analyst/courses/unstructured/
./stagelabs.sh
```

```sh
#!/bin/bash

# Go to the standard location
cd ~/training-data-analyst/courses/unstructured/

# "If at first you don't succeed, try, try again."
#   If this is our first time here, backup the program files
#   If this is a subsequent run, restore fresh from backup before proceeding
#
if [ -d "backup" ]; then
  cp backup/*dataproc* .
else
  mkdir backup
  cp *dataproc* backup
fi

# Verify that the environment variables exist
#
OKFLAG=1
if [[ -v $BUCKET ]]; then
  echo "BUCKET environment variable not found"
  OKFLAG=0
fi
if [[ -v $DEVSHELL_PROJECT_ID ]]; then
  echo "DEVSHELL_PROJECT_ID environment variable not found"
  OKFLAG=0
fi
if [[ -v $APIKEY ]]; then
  echo "APIKEY environment variable not found"
  OKFLAG=0
fi


if [ OKFLAG==1 ]; then
  # Edit the script files
  sed -i "s/your-api-key/$APIKEY/" *dataprocML.py
  sed -i "s/your-project-id/$DEVSHELL_PROJECT_ID/" *dataprocML.py
  sed -i "s/your-bucket/$BUCKET/" *dataprocML.py

  # Copy python scripts to the bucket
  gsutil cp *dataprocML.py gs://$BUCKET/

  # Copy data to the bucket
  gsutil cp gs:\/\/cloud-training\/gcpdei\/road* gs:\/\/$BUCKET\/sampledata\/ 
  gsutil cp gs:\/\/cloud-training\/gcpdei\/time* gs:\/\/$BUCKET\/sampledata\/

fi
```

## Task 3: Natural Language Processing

```sh
cd ~/training-data-analyst/courses/unstructured/
cat 01-dataprocML.py 
```

```python
#!/usr/bin/env python
# Copyright 2018 Google Inc.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

'''
  This program takes a sample text line of text and passes to a Natural Language Processing
  services, sentiment analysis, and processes the results in Python.
  
'''

import logging
import argparse
import json

import os
from googleapiclient.discovery import build

from pyspark import SparkContext
sc = SparkContext("local", "Simple App")

'''
You must set these values for the job to run.
'''
APIKEY="AIzaSyBIANiNuTT3jtT5nzzGXW8gZVLyI_sEQmg"   # CHANGE
print(APIKEY)
PROJECT_ID="qwiklabs-gcp-c742fc67df02460a"  # CHANGE
print(PROJECT_ID) 
BUCKET="qwiklabs-gcp-c742fc67df02460a"   # CHANGE


## Wrappers around the NLP REST interface

def SentimentAnalysis(text):
    from googleapiclient.discovery import build
    lservice = build('language', 'v1beta1', developerKey=APIKEY)

    response = lservice.documents().analyzeSentiment(
        body={
            'document': {
                'type': 'PLAIN_TEXT',
                'content': text
            }
        }).execute()
    return response

## main

sampleline = 'There are places I remember, all my life though some have changed.'
#

# Calling the Natural Language Processing REST interface
#
results = SentimentAnalysis(sampleline)

# 
#  What is the service returning?
#
print("Function returns: ", type(results))

print(json.dumps(results, sort_keys=True, indent=4))

```

## Task 4: Load Sample Data

```sh
gsutil cp /training/road-not-taken.txt gs://$BUCKET/sampledata/road-not-taken.txt
```

## Task 5: Testing Sentiment Analysis with Spark

```sh
cat 02-dataprocML.py
```

```python
#!/usr/bin/env python
# Copyright 2018 Google Inc.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

'''
  This program reads a text file and passes to a Natural Language Processing
  service, sentiment analysis, and processes the results in Spark.
  
'''

import logging
import argparse
import json

import os
from googleapiclient.discovery import build

from pyspark import SparkContext
sc = SparkContext("local", "Simple App")

'''
You must set these values for the job to run.
'''
APIKEY="AIzaSyBIANiNuTT3jtT5nzzGXW8gZVLyI_sEQmg"   # CHANGE
print(APIKEY)
PROJECT_ID="qwiklabs-gcp-c742fc67df02460a"  # CHANGE
print(PROJECT_ID) 
BUCKET="qwiklabs-gcp-c742fc67df02460a"   # CHANGE


## Wrappers around the NLP REST interface

def SentimentAnalysis(text):
    from googleapiclient.discovery import build
    lservice = build('language', 'v1beta1', developerKey=APIKEY)

    response = lservice.documents().analyzeSentiment(
        body={
            'document': {
                'type': 'PLAIN_TEXT',
                'content': text
            }
        }).execute()
    return response

## main

# We could use sc.textFiles(...)
#
#   However, that will read each line of text as a separate object.
#   And using the REST API to NLP for each line will rapidly exhaust the rate-limit quota 
#   producing HTTP 429 errors
#
#   Instead, it is more efficient to pass an entire document to NLP in a single call.
#
#   So we are using sc.wholeTextFiles(...)
#
#      This provides a file as a tuple.
#      The first element is the file pathname, and second element is the content of the file.
#
sample = sc.wholeTextFiles("gs://{0}/sampledata/road-not-taken.txt".format(BUCKET))

# Calling the Natural Language Processing REST interface
#
rdd1 = sample.map(lambda x: SentimentAnalysis(x[1]))


rdd2 =  rdd1.flatMap(lambda x: x['sentences'] )\
            .flatMap(lambda x: [(x['sentiment']['magnitude'], x['sentiment']['score'], [x['text']['content']])] )

  
results = rdd2.take(50)



for item in results:
  print('Magnitude= ',item[0],' | Score= ',item[1], ' | Text= ',item[2])
```
## Task 6: Doing Something Useful

```sh
cat 03-dataprocML.py
```

```python
#!/usr/bin/env python
# Copyright 2018 Google Inc.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

'''
  This program reads a text file and passes to a Natural Language Processing
  service, sentiment analysis, and processes the results in Spark.
  
'''

import logging
import argparse
import json

import os
from googleapiclient.discovery import build

from pyspark import SparkContext
sc = SparkContext("local", "Simple App")

'''
You must set these values for the job to run.
'''
APIKEY="AIzaSyBIANiNuTT3jtT5nzzGXW8gZVLyI_sEQmg"   # CHANGE
print(APIKEY)
PROJECT_ID="qwiklabs-gcp-c742fc67df02460a"  # CHANGE
print(PROJECT_ID) 
BUCKET="qwiklabs-gcp-c742fc67df02460a"   # CHANGE


## Wrappers around the NLP REST interface

def ccccccibkirgntrunuvichtuuutfeucvfrigcrkictge
(text):
    from googleapiclient.discovery import build
    lservice = build('language', 'v1beta1', developerKey=APIKEY)

    response = lservice.documents().analyzeSentiment(
        body={
            'document': {
                'type': 'PLAIN_TEXT',
                'content': text
            }
        }).execute()
    
    return response

## main

# We could use sc.textFiles(...)
#
#   However, that will read each line of text as a separate object.
#   And using the REST API to NLP for each line will rapidly exhaust the rate-limit quota 
#   producing HTTP 429 errors
#
#   Instead, it is more efficient to pass an entire document to NLP in a single call.
#
#   So we are using sc.wholeTextFiles(...)
#
#      This provides a file as a tuple.
#      The first element is the file pathname, and second element is the content of the file.
#
sample = sc.wholeTextFiles("gs://{0}/sampledata/time-machine.txt".format(BUCKET))

# Calling the Natural Language Processing REST interface
#
# results = SentimentAnalysis(sampleline)
rdd1 = sample.map(lambda x: SentimentAnalysis(x[1]))

# The RDD contains a dictionary, using the key 'sentences' picks up each individual sentence
# The value that is returned is a list. And inside the list is another dictionary
# The key 'sentiment' produces a value of another list.
# And the keys magnitude and score produce values of floating numbers. 
#

rdd2 =  rdd1.flatMap(lambda x: x['sentences'] )\
            .flatMap(lambda x: [(x['sentiment']['magnitude'], x['sentiment']['score'], [x['text']['content']])] )

# First item in the list tuple is magnitude
# Filter on only the statements with the most intense sentiments
#
rdd3 =  rdd2.filter(lambda x: x[0]>.75)


results = sorted(rdd3.take(50))


print('\n\n')
for item in results:
  print('Magnitude= ',item[0],' | Score= ',item[1], ' | Text= ',item[2],'\n')
```

## Task 7: Other ML Services
```sh
```
## Module 4 Quiz

### Which (one) of these is NOT a good use case for a ML API?

* Read scanned receipts
* Transcribe support conversations
* __Identify images where your product is shown upside down__
* Identify scenes in a video library where there are aircraft
