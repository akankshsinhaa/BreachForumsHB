# BreachForumsHB
BreachForumsHB

from urllib.parse import urlencode
from selenium import webdriver
from selenium.webdriver.common.by import By
import time
import requests
from bs4 import BeautifulSoup
from datetime import datetime
import os
#import json

from dotenv import load_dotenv
load_dotenv()

API_KEY = os.getenv('SCRAPEOPS_API_KEY')

def get_scrapeops_url(url):
    payload = {'api_key': API_KEY, 'url': url, 'bypass':'cloudflare'}
    proxy_url = 'https://proxy.scrapeops.io/v1/?' + urlencode(payload)
    return proxy_url

searchKeyWord = "database"
def create_folder(folder_name):
    try:
        os.mkdir(folder_name)
        print(f"Folder '{folder_name}' created successfully!")
    except FileExistsError:
        print(f"Folder '{folder_name}' already exists!")

def getPageNum(url):
    response = requests.get(url)

    if response.status_code == 200:
        soup = BeautifulSoup(response.text, 'html.parser')
        
        page_number = soup.find_all('span', class_='pages')
        
        for page in page_number:
            pages = page.text
    else:
        print('Failed to retrieve the web page. Status code:', response.status_code)
    time.sleep(5)
    return pages

try:
    print("Initiating Firefox browser.....")
    fireFoxOptions = webdriver.FirefoxOptions()
    #fireFoxOptions.add_argument("--headless")
    browser = webdriver.Firefox(options=fireFoxOptions)

    main_url = os.getenv('BREACHFORUMS_URL')
    url = main_url+'/Forum-Databases'

    print("Navigating to requested page.....")
    browser.get(url)
    time.sleep(60)
    pages = browser.find_element(By.CLASS_NAME, 'pages')
    
    pages = pages.text
    pages = int(pages[pages.index("(") + 1:pages.index(")")])
    print("Scraping each page.....")
    latest = []
    for page_num in range(1,pages+1):
        if(page_num==1):
            currpage = f"{url}"
        else:
            currpage = f"{url}?page={page_num}"
        print(f"Currently scraping page number {page_num}...")
        browser.get(currpage)
        soup = BeautifulSoup(browser.page_source, 'html.parser')
        texts = soup.find_all('a')
        for a_tag in texts:
            text = a_tag.text.strip()
            text = text.lower()
            href = a_tag.get('href', '')
            if(text.find(searchKeyWord)>0):
                print(f"Found '{searchKeyWord}' at {main_url}/{href}")
                latest.append(f"""{main_url}/{href}""")
    print("Adding details to JSON files....")    
    for x in latest:
        now = datetime.now()
        time.sleep(1)
        dt_string_text = now.strftime("%d-%m-%Y-%H-%M-%S")
        file_name = str('result-'+dt_string_text+'.json')
        browser.get(x)
        post_images = browser.find_elements(By.CLASS_NAME,"mycode_img")
        k = 0
        image_list = []
        print(f"Creating dictionary with JSON info for{x}....")
        post_content = browser.find_elements(By.CSS_SELECTOR,".post_body")
        post_text = ""
        info_dict = {
                        "link" : "",
                        "content" : "",
                        "image" : []
                    }
        for post in post_content:
            post_text += post.text
        info_dict["link"] = x.replace("'", '')
        info_dict["content"] = post_text.replace("'", '')
        src = []
        print("Extracting images...")
        for image in post_images:
            src.append(image.get_attribute('src'))
        print("Saving images....")
        for image in src:
            data = requests.get(image).content
            image_without_slash = image.replace("/","")
            f = open(f'img-{image_without_slash}.jpg','wb')
            f.write(data) 
            f.close() 
        info_dict["image"] = src
        info_dict = str(info_dict).replace("'",'"')
        print("Writing to file...")
        with open(file_name, 'w') as file:
            file.write(info_dict)

finally:
    try:
        browser.close()
        browser.quit()
    except:
        pass
