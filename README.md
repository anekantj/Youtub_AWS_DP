# Youtub_Project_220525

The dataset is take from kaggle wbesite which contains raw data in the format of csv and json.The dataset is around the youtube videos properties like name id title likes etc.
Since AWS cralwer was unable to scan the data from the csv and json file we had to cleanse the data before converting the data format as paruet for standardization purposes.

CSV-->CSV files with stable encoding.
Json--> Json files which are normalized and does not contain nested json.

Transformations: Once the files were properly converted we perfmored joining operation between the two table using ETL Visual Job in AWS Glue.

This transformed data was stored into another bucket on top of which we can run a crawler to get meta data and create an specific table on it.

Volumne of data in kaggle set: 600 Mb
Final volumne:250 MB
