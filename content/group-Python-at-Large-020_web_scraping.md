---
Title: Web-scraping of SEC EDGAR 10-K filings (Group Python at Large)
Date: 10 April, 2022
Category: Progress Report
---

By Group "Python at Large"

The first step of our group project involves extracting the textual data from Item 7. Management's Discussion and Analysis, and Notes to consolidated financial in Item 8. Financial Statements and Supplementary Data of SEC 10-K file.    
According to SEC EDGAR, there is an Extractor API that enables easy extraction of the sections needed from 10-K files.  
An example of the code we use is as follows:
```python
from sec_api import ExtractorApi
extractorApi = ExtractorApi("API Key") #need to get the 'API Key' from https://sec-api.io/
# the 10-K url of VEEVA SYSTEMS INC
filing_url = "https://www.sec.gov/Archives/edgar/data/0001393052/000156459018007164/veev-10k_20180131.htm"
# get the original text of Item 7
section_text = extractorApi.get_section(filing_url, "7", "text")
print(section_text)

```
Although the API is a very handy tool, SEC only allows a limited number of extraction for each free API key. We then have to find other ways to download and extract the textual data needed.     
To download the 10-K filings from SEC EDGAR, we made several different tryouts.

### Tryout 1: use 'request' package

#### Step 1: Download the quarterly index files using [edgar](https://pypi.org/project/python-edgar/) package.

The code we use for this step is as follows:  
```python
def get_index_files(start_year: int = str,
                   store_path = str,
                   user = str):
    '''
    Download the crude index files from sec-edgar
    :param start_year: int, the starting year of indices that we want (2014 in our case)
    :param store_path: str, the path to store the crude index file
    :param user: str, username to download index file, usually company email (hku student could simply use the @connect.hku.hk email)
    :return: none, but index file will be downloaded and stored in store_path
    '''
    ed.download_index(store_path, start_year, user, skip_all_present_except_last=False)

# sample program to use the function
get_index_files(start_year = 2016, store_path = './', user = 'YourUID@connect.hku.hk')
```

#### Step 2: Clean and merge all index files to get a complete list during the specified period

The code we use for this step is as follows:  
```python
def clean_index_files(start_year: int,
                      end_year: int,
                      read_path: str):
    '''
    Get the complete and clean index list from the crude index file
    :param start_year: int, starting year of 10K reports we want (2016 in our case)
    :param end_year: int, ending year of 10K reports we want (2020 in our case)
    :param read_path: str, the path to load the crude index file from (same as the store_path in Step 1)
    :return: DataFrame, the complete index file
    '''
    indices = pd.DataFrame()
    for i in range(start_year,end_year+1): # for each year's indices
        # load the indices of all quarters
        q1 = pd.read_csv(str(read_path) + str(i)+'-QTR1.tsv', delimiter="|", header=None)
        q2 = pd.read_csv(str(read_path) + str(i)+'-QTR2.tsv', delimiter="|", header=None)
        q3 = pd.read_csv(str(read_path) + str(i) + '-QTR3.tsv', delimiter="|", header=None)
        q4 = pd.read_csv(str(read_path) + str(i) + '-QTR4.tsv', delimiter="|", header=None)
        m = pd.concat([q1,q2,q3,q4],ignore_index=True)
        m.columns = ['CIK', 'Comname', 'File_Type', 'File_Date', 'txt_path', 'html_path']
        # keep only 10K reports
        m = m[m['File_Type']=='10-K']
        m = m.sort_values(by=['CIK','File_Date'])
        m.index = range(len(m.index))
        # merge indices of all quarters in all years together to get the complete index list
        indices = pd.concat([indices,m],ignore_index=True)
    # add the txt_name column for later use
    indices['txt_name'] = indices['txt_path'].apply(lambda x: x.split('/')[-2] + x.split('/')[-1])
    return indices

# sample program to use the function
indices = clean_index_files(start_year=2016,end_year=2020,read_path='./')
```

#### Step 3: Download 10-K txt files according to the index file using requests

The code we use for this step is as follows:
```python
def get_10K_txt(txt_path: str,
                txt_name: str,
                store_path: str = st.DATA_CRUDE):
    '''
    Get raw 10K txt file and store it in specified directories
    :param txt_path: str, the txt path to download 10K from
    :param txt_name: str, the name of 10K txt file
    :param store_path: str, path to store crude 10K txt file
    :return: none, but 10K txt file will be stored in specified directories
    '''

    # get the desired 10K report by requests
    r = requests.get('https://www.sec.gov/Archives/' + txt_path, headers={'User-Agent': 'Mozilla/5.0'})

    if not os.path.exists(store_path):
        os.mkdir(store_path)

    # store the 10K into specified path
    f = open(store_path + txt_name, 'wb')
    f.write(r.content)
    f.close()
# sample program to use the function
for i in range(len(indices['CIK'])): # download all the 10K reports listed in the cleaned index file
    txt_path = indices.loc[i, 'txt_path']
    txt_name = indices.loc[i, 'txt_name']
    get_10K_txt(txt_path,txt_name)
```

#### Pros:

1. This is the most original way to scrape 10-K files on our own. Instead of relying on additional database for accessing the 10-K reports, we download them using 'requests' directly.
2. The 10-K reports downloaded are completely raw, so we could make any further conversion or calculation on our own.

#### Cons:

1. This is the most tedious way to obtain 10-K reports, it takes several steps to download them.
2. The index files are directly downloaded and stored in a specified directory rather than loaded as a variable in IDE.
3. The index files downloaded are quarterly, if we want to obtain 10-K across years, we need to merge quarterly files into yearly files.
4. The 10-K reports are raw, taking up a lot of memory space (around 160MB per 10 files). It also takes a lot of time to download them (around 13 seconds per 10 files).


### Tryout 2: use 'secedgar' package

We also find a handy package [secedgar(version 0.4.0a2)](https://sec-edgar.github.io/sec-edgar/index.html) that can scrape all types of SEC filings.

The code we use is as follows:
```python
# import date
from datetime import date
# import nest_asyncio to allow nested use
import nest_asyncio
nest_asyncio.apply()
# import CompanyFilings, FilingType from secedgar for scraping filings from SEC EDGAR
from secedgar import CompanyFilings, FilingType
'''
class CompanyFilings() - Base class for receiving EDGAR filings.
:param cik_lookup: str/list of str, the ticker(s) of concerned firm(s)
:param filing_type: valid filing type enum,
      incorporated in the Union[secedgar.core.filing_types.FilingType],
      (in our case 'FILING_10K')
:param start_date: datetime.date, the starting date of files we need
:param end_date: datetime.date, the ending date of files we need
:param user_agent: str, username to download crude files,
      usually takes the form of 'yourname(company email)'
      (hku student could simply use the @connect.hku.hk email)
'''
cik_list = ["aapl", "fb", "msft"]
# download filings of companies of interests in the cik_list
my_filings = CompanyFilings(cik_lookup = cik_list,
                          filing_type = FilingType.FILING_10K,
                          start_date = date(2016, 1, 1),
                          end_date = date(2020, 12, 31),
                          user_agent = "YourName(YourUID@connect.hku.hk)")
# save the downloaded files into a given path
my_filings.save('./10-K')
```
**For demonstration purpose, we only include three companies in the* `cik_list`.

#### Pros:

1. This is package is easier to use than 'request'. We can directly scrape filings given the time range and the list of tickers of the target companies.
2. The package enables a faster scraping speed (around 4 files per second)

#### Cons:

1. Same as using 'request', the raw text files can only be directly stored in a specified local directory rather than a variable in the IDE.
2. Same as using 'request', the raw 10-K reports take up too much memory storage.
3. It takes longer time to read through each file to clean and extract textual data.


### Tryout 3: download from public data repository contributed by other researchers

We then found a data repository with all cleaned raw 10-K files available from 1993 to 2021.
[The Notre Dame Software Repository for Accounting and Finance (SRAF)](https://sraf.nd.edu/sec-edgar-data/)
is a website designed to provide a central repository for programs and data used in accounting and finance research, with a focus on textual analysis.

For our project, we ended up directly downloading cleaned data files and then extract the concerned sections. However, our previous tryouts act as valuable experience and could be transferrable to analyzing other types of SEC filings that are not directly available in the repository.
