# web_scape_stock_price_qurl
Web scraping local bank stock prices as part of a larger study on banking performance over the course of 5 years as well as getting necessary information to build up a model of stock price in relation to its performance indicators.

""""""""""""""""""""""""""""""""""
SCRAPING WEBSITE
Case: Cafe
Target data: All listed company information 


Libraries requires
    - webdriver: available to download in selenium. It is an API support dynamic web page
    - BeautifulSoup: It is an API support static web page
    - time: delay time execution
    - xlsxwriter: read and write in excel through python
    - pandas: store, manupulate and export to excel
    
""""""""""""""""""""""""""""""""""
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
    Part 1:
    Gather all company names and their website links from cafef data base
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
import sys
!{sys.executable} -m pip install -U selenium
from selenium import webdriver
from selenium.webdriver.support.ui import Select
from selenium.webdriver.common.keys import Keys
from selenium.common.exceptions import NoSuchElementException
import time


# os provides a portable way of using operating system dependent functionality.
import os
os.environ["PATH"] += os.pathsep + r'E:\Download'

# open the website through Chrome using webdriver
driver = webdriver.Chrome()
driver.get('http://s.cafef.vn/du-lieu-doanh-nghiep.chn#data')

# find out all available option for industry selection
options = dict()
for p in driver.find_elements_by_xpath('//*[@id="CafeF_ThiTruongNiemYet_Nganh"]/option'):
    value = p.get_attribute("value")  #extract the value for each option
    text = p.text
    options[value] = text
    
# using industry options
industry_option = Select(driver.find_element_by_id('CafeF_ThiTruongNiemYet_Nganh'))   # the dropdown for industries

#loop through all selected industry options for the code and full name of all listed companies. Noted that we will not choose options: -1    

for value in [k for k,v in options.items() if "-1" == k]:
    # Create a dictionary for all name and their respective code in each industry
    all_code = {}
    
    industry_option.select_by_value(value)
    driver.find_element_by_xpath('//*[@id="CafeF_DSCongtyNiemyet"]/div/div[1]/table/tbody/tr[3]/td/table/tbody/tr[2]/td[3]/img').click()
    time.sleep(5)
    
    # click the 'Expand list' option to see all listed company 
    try:
        driver.find_element_by_xpath('//*[@id="CafeF_ThiTruongNiemYet_Trang"]/a[2]').click()  
    except NoSuchElementException:
        pass
    finally:
        time.sleep(5) 
    
        for z in driver.find_elements_by_xpath('//*[@id="CafeF_ThiTruongNiemYet_Content"]/table/tbody/tr/td/a[@style="font-weight: normal;"]'):
            link = z.get_attribute("href")[18:]
            sublink1, sublink2 = link.split("/")
            subtext1, subtext2 = sublink2.split("-",1)
            all_code[subtext1] = subtext2
            
# Getting all banks
all_bank_code = {k:v for k,v in all_code.items() if "ngan-hang" in v and "cong-ty" not in v and "quy-dau-tu" not in v }

"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
    Part 2:
    We will then use BeautifulSoup to fetch their financial data

"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
import requests
from bs4 import BeautifulSoup
import re
import pandas as pd

# Set report type

for report in ['BS', 'PnL']:
    if report == "PnL": 
        report_code = "IncSta"
        report_name = "ket-qua-hoat-dong-kinh-doanh"
    else:
        report_code = "BSheet"
        report_name = "bao-cao-tai-chinh"     
   
    # Set time       
    for year in range(2019,2020): 
        indicator = []
        firm_code = []
        firm_name = []
        industry  = []
        report_type = []
        locals()['t_q2%d'%(year-0)] = []
        locals()['t_q1%d'%(year-0)] = []
        locals()['t_q3%d'%(year-1)] = []
        locals()['t_q4%d'%(year-1)] = []
        
    # Set loop for both keys and values in the dictionary   "344" == k or "345" == k or "346" == k
        for value in [k for k,v in options.items() if "341" == k ]:
            #for code, name in locals()['part_%s'%(value)].items():
            for code, name in all_bank_code.items():
                
                # query the website and return the html to the vaiable `page'
                page = requests.get('http://s.cafef.vn/bao-cao-tai-chinh/'+ code + '/%s/%d/4/0/0/%s-'%(report_code, year, report_name) + name)  
                time.sleep(10)
                
                # create a BeautifulSoup object, or a parse tree.
                soup = BeautifulSoup(page.content, 'html.parser')
                   
                # create a list. A list is an ordered sequence of elements re.compile(r'^((?!display:none).)*$')
                all = []

                for list in soup.find_all("tr",attrs={'id':True}) :
                    for item_name in list.find_all("td",attrs={'style':True,'align': None}) :
                        text = (item_name.get_text(strip=True)) 
                        indicator.append(text)
                    for item_name in list.find_all('td', attrs={'style':True,'align':True}):
                        text = (item_name.get_text(strip=True)) 
                        all.append(text)
                        
               
                for num in [i for i in range(0,len(all),4)]:
                    locals()['t_q4%d'%(year-1)].append(all[num])       
                for num in [i for i in range(1,len(all),4)]:
                    locals()['t_q3%d'%(year-1)].append(all[num])  
                for num in [i for i in range(2,len(all),4)]:
                    locals()['t_q1%d'%(year-0)].append(all[num])  
                for num in [i for i in range(3,len(all),4)]:
                    locals()['t_q2%d'%(year-0)].append(all[num]) 
                    

                # Getting firm name, code and industry name 
                item = soup.find("a",{'href':True})
                text = item.get_text(strip=True)
            
                for _ in range(int(len(all)/4)):
                    firm_name.append(text)
                    firm_code.append(code)
                    report_type.append(report)
                    industry.append('Banks')
                else:
                    pass
        
        locals()['df_%s_%d'%(report, year)] = pd.DataFrame({'Industry': industry, 'Report': report_type, 'Firm Code': firm_code, 'Firm Name': firm_name, 'Indicator': indicator, 'q3%d'%(year-1): locals()['t_q3%d'%(year-1)], 'q4%d'%(year-1) : locals()['t_q4%d'%(year-1)], 'q1%d'%(year-0) : locals()['t_q1%d'%(year-0)], 'q2%d'%(year-0) : locals()['t_q2%d'%(year-0)] })
        
        locals()['df_%s_%d'%(report, year)] =locals()['df_%s_%d'%(report, year)][~locals()['df_%s_%d'%(report, year)]['Indicator'].str.contains('-')]
        
# Merge and manipulate dataframe 


for year in range(2019,2020):     
    df_all = pd.concat([locals()['df_BS_%d'%(year)], locals()['df_PnL_%d'%(year)]])

    # Create a Pandas Excel writer using XlsxWriter as the engine.
    writer = pd.ExcelWriter('E:\\Python\\results\\All_bank_data_quarterly %d.xlsx'%(year), engine='xlsxwriter')

    # Convert the dataframe to an XlsxWriter Excel object.
    df_all.to_excel(writer, sheet_name='All')

    # Close the Pandas Excel writer and output the Excel file.
    writer.save()

df_BS_2019 = df_BS_2019.drop(["q32018", "q42018"], axis=1)
df_PnL_2019 = df_PnL_2019.drop(["q32018", "q42018"], axis=1)
df_BS_2019 = df_BS_2019.rename(columns={"q42019": "q12019"})
df_PnL_2019  = df_PnL_2019.rename(columns={"q42019": "q12019"})

df_all = pd.concat([df_BS_2019, df_PnL_2019])    
    
    
    
    
    
    
