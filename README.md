<h1> 1. Exploring natality dataset </h1>

Este Notebook Ilustra:
<ol>
<li> Explorando uma base de dados BigQuery usando AI Platform Notebooks.
</ol>


```python
!sudo chown -R jupyter:jupyter /home/jupyter/training-data-analyst
```


```python
# change these to try this notebook out
BUCKET = 'cloud-training-demos-ml-babies'
PROJECT = 'cloud-training-demos'
REGION = 'us-central1'
```


```python
import os
os.environ['BUCKET'] = BUCKET
os.environ['PROJECT'] = PROJECT
os.environ['REGION'] = REGION
```


```bash
%%bash
if ! gsutil ls | grep -q gs://${BUCKET}/; then
  gsutil mb -l ${REGION} gs://${BUCKET}
fi
```

<h2> Explorando os Dados </h2>

The data is natality data (record of births in the US). My goal is to predict the baby's weight given a number of factors about the pregnancy and the baby's mother.  Later, we will want to split the data into training and eval datasets. The hash of the year-month will be used for that -- this way, twins born on the same day won't end up in different cuts of the data.




```python
# Create SQL query using natality data after the year 2000
query = """
SELECT
  weight_pounds,
  is_male,
  mother_age,
  plurality,
  gestation_weeks,
  FARM_FINGERPRINT(CONCAT(CAST(YEAR AS STRING), CAST(month AS STRING))) AS hashmonth
FROM
  publicdata.samples.natality
WHERE year > 2000
"""
```


```python
# Call BigQuery and examine in dataframe
from google.cloud import bigquery
df = bigquery.Client().query(query + " LIMIT 100").to_dataframe()
df.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>weight_pounds</th>
      <th>is_male</th>
      <th>mother_age</th>
      <th>plurality</th>
      <th>gestation_weeks</th>
      <th>hashmonth</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>7.063611</td>
      <td>True</td>
      <td>32</td>
      <td>1</td>
      <td>37.0</td>
      <td>7108882242435606404</td>
    </tr>
    <tr>
      <th>1</th>
      <td>4.687028</td>
      <td>True</td>
      <td>30</td>
      <td>3</td>
      <td>33.0</td>
      <td>-7170969733900686954</td>
    </tr>
    <tr>
      <th>2</th>
      <td>7.561856</td>
      <td>True</td>
      <td>20</td>
      <td>1</td>
      <td>39.0</td>
      <td>6392072535155213407</td>
    </tr>
    <tr>
      <th>3</th>
      <td>7.561856</td>
      <td>True</td>
      <td>31</td>
      <td>1</td>
      <td>37.0</td>
      <td>-2126480030009879160</td>
    </tr>
    <tr>
      <th>4</th>
      <td>7.312733</td>
      <td>True</td>
      <td>32</td>
      <td>1</td>
      <td>40.0</td>
      <td>3408502330831153141</td>
    </tr>
  </tbody>
</table>
</div>



Let's write a query to find the unique values for each of the columns and the count of those values.
This is important to ensure that we have enough examples of each data value, and to verify our hunch that the parameter has predictive value.


```python
# Create function that finds the number of records and the average weight for each value of the chosen column
def get_distinct_values(column_name):
  sql = """
SELECT
  {0},
  COUNT(1) AS num_babies,
  AVG(weight_pounds) AS avg_wt
FROM
  publicdata.samples.natality
WHERE
  year > 2000
GROUP BY
  {0}
  """.format(column_name)
  return bigquery.Client().query(sql).to_dataframe()
```


```python
# Bar plot to see is_male with avg_wt linear and num_babies logarithmic
df = get_distinct_values('is_male')
df.plot(x='is_male', y='num_babies', kind='bar');
df.plot(x='is_male', y='avg_wt', kind='bar');
```


    
![png](output_10_0.png)
    



    
![png](output_10_1.png)
    



```python
# Line plots to see mother_age with avg_wt linear and num_babies logarithmic
df = get_distinct_values('mother_age')
df = df.sort_values('mother_age')
df.plot(x='mother_age', y='num_babies');
df.plot(x='mother_age', y='avg_wt');
```


    
![png](output_11_0.png)
    



    
![png](output_11_1.png)
    



```python
# Bar plot to see plurality(singleton, twins, etc.) with avg_wt linear and num_babies logarithmic
df = get_distinct_values('plurality')
df = df.sort_values('plurality')
df.plot(x='plurality', y='num_babies', logy=True, kind='bar');
df.plot(x='plurality', y='avg_wt', kind='bar');
```


    
![png](output_12_0.png)
    



    
![png](output_12_1.png)
    



```python
# Bar plot to see gestation_weeks with avg_wt linear and num_babies logarithmic
df = get_distinct_values('gestation_weeks')
df = df.sort_values('gestation_weeks')
df.plot(x='gestation_weeks', y='num_babies', logy=True, kind='bar');
df.plot(x='gestation_weeks', y='avg_wt', kind='bar');
```


    
![png](output_13_0.png)
    



    
![png](output_13_1.png)
    


All these factors seem to play a part in the baby's weight. Male babies are heavier on average than female babies. Teenaged and older moms tend to have lower-weight babies. Twins, triplets, etc. are lower weight than single births. Preemies weigh in lower as do babies born to single moms. In addition, it is important to check whether you have enough data (number of babies) for each input value. Otherwise, the model prediction against input values that doesn't have enough data may not be reliable.
<p>
In the next notebook, I will develop a machine learning model to combine all of these factors to come up with a prediction of a baby's weight.

Copyright 2017-2018 Google Inc. Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with the License. You may obtain a copy of the License at http://www.apache.org/licenses/LICENSE-2.0 Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License
