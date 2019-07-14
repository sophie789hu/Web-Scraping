# web-scraping

Thanks to the Web Scraping technique, data are more than accessible.
This technique (and the powerful Beautiful Soup package) allows to automatically extract large amounts of data from a website.
This exercice shows how it can to get a list of Universities from an up-to-date database maintained by Klaus Förster - univ.cc.

Python version 7.3.7

'''

from bs4 import BeautifulSoup
from urllib.request import Request, urlopen
import pandas as pd
import re
import datetime
import xlsxwriter


def WorldScraping():
        """
        Take all World page urls
        """

        world_url = "https://univ.cc/world.php"

        world_req = Request(world_url, headers={'User-Agent': 'Mozilla/5.0'})
        world_webpage = urlopen(world_req).read()
        world_soup = BeautifulSoup(world_webpage, 'html.parser')

        world_all_url = list()
        world_country = list()
        for id in range(1, len(world_soup.tr.find_all("option"))):
                #get url on the first page
                country_id1 = world_soup.tr.find_all("option")[id]["value"]
                country_name1 = world_soup.tr.find_all("option")[id].text
                country_url1 = "https://univ.cc/search.php?dom="+country_id1+"&key=&start=1"
                world_all_url.append(country_url1)
                #check if not other pages for one country
                country_webpage = urlopen(country_url1).read()
                country_soup = BeautifulSoup(country_webpage, 'html.parser')
                if country_soup.find_all("nav", class_="resultNavigation"):
                        result = range(len(country_soup.find_all("nav", class_="resultNavigation"))-1)
                        for soup in result:
                                print(soup)
                                country_id2 = country_soup.nav.find_all("a", href=True)[soup]["href"]
                                country_url2 = "https://univ.cc/"+country_id2
                                world_all_url.append(country_url2)
                world_country.append(country_name1)

        world_df = pd.DataFrame({"Country/State":world_country, "url":world_all_url})
        world_df["Source"] = "World"

        return world_df


def UsaScraping():
        """
        Take all USA page urls
        """

        usa_url = "https://univ.cc/states.php"

        usa_req = Request(usa_url, headers={'User-Agent': 'Mozilla/5.0'})
        usa_webpage = urlopen(usa_req).read()
        usa_soup = BeautifulSoup(usa_webpage, 'html.parser')

        usa_all_url = list()
        usa_state = list()
        for id in range(1, len(usa_soup.tr.find_all("option"))):
                #get url on the first page
                state_id1 = usa_soup.tr.find_all("option")[id]["value"]
                state_name1 = usa_soup.tr.find_all("option")[id].text
                state_url1 = "https://univ.cc/search.php?dom="+state_id1+"&key=&start=1"
                usa_all_url.append(state_url1)
                #check if not other pages for one state
                state_webpage = urlopen(state_url1).read()
                state_soup = BeautifulSoup(state_webpage, 'html.parser')
                if state_soup.find_all("nav", class_="resultNavigation"):
                        result = range(len(state_soup.find_all("nav", class_="resultNavigation"))-1)
                        for soup in result:
                                state_id2 = state_soup.nav.find_all("a", href=True)[soup]["href"]
                                state_url2 = "https://univ.cc/"+state_id2
                                usa_all_url.append(state_url2)
                usa_state.append(state_name1) 

        usa_df = pd.DataFrame({"Country/State":usa_state, "url":usa_all_url})
        usa_df["Source"] = "USA"

        return usa_df


def WrapperData(world_df, usa_df):
        """
        Append all url from World and USA together
        """

        all_url = world_df.append(usa_df)
        removeParathense = re.compile("\s\((.*?)\)") #remove parenthenses and what is inside
        all_url["Country/State"] = all_url["Country/State"].str.replace(removeParathense, "", regex=True)
        all_url.reset_index(drop=True, inplace=True)

        #Take all University name and Url
        all_uni = pd.DataFrame([])
        uni_source = list()
        uni_country_state = list()
        uni_name = list()
        uni_link = list()

        for url in range(len(all_url["url"])):
        #enter in the url
                webpage = urlopen(all_url["url"][url]).read()
                soup = BeautifulSoup(webpage, 'html.parser')
                #append by uni: source, country/state, name and link
                source = pd.Series(map(lambda index: all_url["Source"][url], range(len(soup.find_all("li")))))
                uni_source.extend(source)
                country_state = pd.Series(map(lambda index: all_url["Country/State"][url], range(len(soup.find_all("li")))))
                uni_country_state.extend(country_state)
                name = pd.Series(map(lambda index: soup.ol.find_all("li")[index].a.text, range(len(soup.find_all("li")))))
                uni_name.extend(name)
                link = pd.Series(map(lambda index: soup.ol.find_all("li")[index].a.attrs["href"], range(len(soup.find_all("li")))))
                uni_link.extend(link)

        all_uni = pd.DataFrame({"Source":uni_source, 
                                "Country/State":uni_country_state,
                                "University name":uni_name,
                                "University url":uni_link
                                })

        all_uni.sort_values("Country/State", ascending=False)

        return all_uni

def ExtractExcel(all_uni):
        """
        Extract all uni data into Excel file
        """
        today = datetime.now().strftime("%Y_%m_%d_%H-%M")
        fileName = "list of all university in the world"+today+".xlsx"            
        writer = pd.ExcelWriter(fileName, engine='xlsxwriter')
        all_uni.to_excel(writer, sheet_name='Universities', index=False)
        writer.save()



world_df = WorldScraping()
usa_df = UsaScraping()
all_uni = WrapperData(world_df, usa_df)
ExtractExcel(all_uni)
'''
