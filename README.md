# 📰 Web Scraping Tool for Andhrajyothy.com

## Overview 📋

This Python-based web scraping tool captures the latest news and articles from "andhrajyothy.com" across various categories, saving each category's articles in structured JSON files. Designed with efficient scraping methods, this tool enables seamless data gathering using BeautifulSoup, Requests, and custom error-handling techniques.

---

## Requirements 🛠️

- **Python Version**: 3.7 or higher
- **Libraries**:
  - `requests` 📥 - For fetching webpage content.
  - `beautifulsoup4` 🥣 - For HTML parsing and extracting article data.
  - `json` 📦 - For saving and managing scraped content.

---

## Virtual Environment Setup 🌐

1. **Create a Virtual Environment**:
   ```bash
   conda create -n myenv python=3.7
   ```
2. Activate the environment:
   ```bash
   conda activate myenv
   ```
3. Install the required packages:
   ```bash
   pip install requests beautifulsoup4
   ```
---

## Workflow 🚀

1. Load Previously Scraped Links to avoid duplicate scraping.
2. Scrape Article Links page-by-page for each category.
3. Extract and Process Each Article: Titles, URLs, categories, and content.
4. Save Data: Each category is saved as a unique JSON file once all articles are scraped.
5. Dynamic Content Filtering: Avoids redundant scraping and updates completed entries.
   
---

## Sample Output 🖥️

```json
[
    {
        "title": "Sample News Title",
        "href": "https://www.andhrajyothy.com/sample-news-article",
        "category": "news",
        "content": "Paragraph 1 of the article. Paragraph 2 of the article."
    },
    ...
]
```
---

### Technical Highlights 🔍

#### HTML Tag Extraction

- **Anchor Tag (`<a>`)**: Defines hyperlinks and links to article pages.
  
- **Paragraph Tag (`<p>`)**: Extracts article content.
  
### Step-by-Step Process

#### Loading Previously Scraped Links
```python
def load_scraped_hrefs():
    try:
        with open('scraped_hrefs.json', 'r', encoding='utf-8') as f:
            return set(json.load(f))
    except FileNotFoundError:
        return set()
```

#### Saving Scraped Links
```python
def save_scraped_hrefs(hrefs):
    with open('scraped_hrefs.json', 'w', encoding='utf-8') as f:
        json.dump(list(hrefs), f, ensure_ascii=False, indent=4)
```

#### Getting Article Links
```python
def get_article_links(url):
    headers = {'User-Agent': choice(user_agents)}
    for attempt in range(5):
        try:
            response = requests.get(url, headers=headers)
            response.raise_for_status()
            soup = BeautifulSoup(response.content, 'html.parser')
            article_links = soup.select('a[href]')
            links = [link['href'] for link in article_links if 'article' in link['href']]
            return links
        except requests.RequestException as e:
            print(f"Attempt {attempt + 1} failed: {e}")
            time.sleep(2)
    return []
```

#### Loading Previously Scraped Links (Alternate)
```python
def load_scraped_links(filename="scraped_hrefs.json"):
    try:
        with open(filename, 'r', encoding='utf-8') as file:
            return json.load(file)
    except FileNotFoundError:
        return []
```

#### Saving Scraped Link
```python
def save_scraped_link(link, filename="scraped_hrefs.json"):
    scraped_links = load_scraped_links(filename)
    if (link not in scraped_links):
        scraped_links.append(link)
        with open(filename, 'w', encoding='utf-8') as file:
            json.dump(scraped_links, file, ensure_ascii=False, indent=4)
```

#### Making HTTP Requests
```python
def make_request(url, max_retries=5):
    session = requests.Session()
    retry_strategy = Retry(
        total=max_retries,
        backoff_factor=1,
        status_forcelist=[429, 500, 502, 503, 504],
        allowed_methods=["HEAD", "GET", "OPTIONS"]
    )
    adapter = HTTPAdapter(max_retries=retry_strategy)
    session.mount('http://', adapter)
    session.mount('https://', adapter)
    headers = get_headers()
    try:
        response = session.get(url, headers=headers)
        response.raise_for_status()
        return response
    except requests.exceptions.RequestException as e:
        print(f"Error for URL {url}: {e}")
        return None
```

#### Scraping a Page
```python
def scrape_page(url, category):
    response = make_request(url)
    if response and response.status_code == 200:
        print(f"Successfully retrieved page: {url}")
        soup = BeautifulSoup(response.content, 'html.parser')
        article_blocks = soup.select('body > div.container-fluid > div > div.categoryPage_Wrapper > div.leftSidebar > div > div.category_content > div.category_content_listing > figure > figcaption > h3 > a')
        content_selector = get_selector(category)

        print(f"Found {len(article_blocks)} articles on page: {url}")

        articles_found = False
        for a_tag in article_blocks:
            href = a_tag['href']
            full_href = 'https://www.andhrajyothy.com/' + href if not href.startswith('http') else href
            title = a_tag.text.strip()

            article_response = make_request(full_href)
            if article_response and article_response.status_code == 200:
                article_soup = BeautifulSoup(article_response.content, 'html.parser')
                content_element = article_soup.select_one(content_selector)
                if content_element:
                    content = content_element.text.strip()
                else:
                    content_elements = article_soup.select('p')
                    content = ' '.join([elem.text for elem in content_elements]).strip() if content_elements else 'Content not found'

                print(f"Found article: {title} - {full_href}")
                article = {'title': title, 'href': full_href, 'content': content, 'category': category}
                save_scraped_link(full_href)
                append_article(article, category)
                articles_found = True
            else:
                print(f"Failed to retrieve article page: {full_href}")

        return articles_found
    else:
        print(f"Failed to retrieve page: {url}")
        return False
```

#### Scraping All Pages
```python
def scrape_all_pages(base_url, category):
    page = 1
    while True:
        url = base_url if page == 1 else f"{base_url}/page/{page}/"
        print(f"Scraping page: {url}")
        articles_found = scrape_page(url, category)
        if not articles_found:
            break
        page += 1
        time.sleep(randint(1, 3))
```

### Main Execution
```python
if __name__ == "__main__":
    categories = [
        {'url': 'https://www.andhrajyothy.com/national', 'category': 'national'},
        {'url': 'https://www.andhrajyothy.com/latest-news', 'category': 'latest_news'},
        {'url': 'https://www.andhrajyothy.com/elections', 'category': 'elections'},
        {'url': 'https://www.andhrajyothy.com/andhra-pradesh', 'category': 'andhra_pradesh'},
        {'url': 'https://www.andhrajyothy.com/telangana', 'category': 'telangana'},
        {'url': 'https://www.andhrajyothy.com/sports', 'category': 'sports'},
        {'url': 'https://www.andhrajyothy.com/navya', 'category': 'navya'},
        {'url': 'https://www.andhrajyothy.com/editorial', 'category': 'editorial'},
        {'url': 'https://www.andhrajyothy.com/business', 'category': 'business'},
        {'url': 'https://www.andhrajyothy.com/politics', 'category': 'politics'},
        {'url': 'https://www.andhrajyothy.com/mukhyaamshalu', 'category': 'mukhyaamshalu'},
    ]

    for category in categories:
        print(f"Scraping category: {category['url']}")
        scrape_all_pages(category['url'], category['category'])
```
---
### Key Challenges & Solutions 💡

- **Handling Dynamic Content**:
  - **Challenge**: Some articles load via JavaScript, limiting scraping accuracy.
  - **Solution**: Applied alternative HTML tag selection methods to reliably access content.

- **Reducing Latency**:
  - **Challenge**: Ensuring the scraper doesn’t get blocked while optimizing speed.
  - **Solution**: Randomized user-agents and implemented retry logic for seamless request handling.

- **Data Management**:
  - **Challenge**: Maintaining data integrity with frequent script updates.
  - **Solution**: Created JSON backups for each category, ensuring data persistence.

---

### Additional Insights 📊
This project showcases a robust and scalable approach to web scraping using Python, prioritizing both reliability and adaptability. The use of reusable functions streamlines the codebase, promoting easier maintenance and updates. Flexible tag selection enables the tool to extract data from various HTML structures, making it versatile for different websites or content types. Dynamic error handling ensures that the scraper can gracefully manage common issues such as network errors or changes in website structure, allowing for a smoother user experience. Overall, this tool is well-equipped for efficiently scraping articles and other web content, making it a valuable asset for data collection projects.

### Future Enhancements 🎯
- **Pagination Limits**: Currently, the script scrapes all available pages within each category for comprehensive coverage. Adding pagination limits would let users specify a range of pages, targeting specific timeframes. This would enhance efficiency in data processing and storage, especially for larger websites.
By allowing defined start and end pages, the script can optimize data collection.

- **Additional Data Cleaning**: Existing techniques focus on basic content extraction and formatting. Implementing advanced filters and normalization processes would improve content quality. This could involve removing duplicates, applying keyword filters, and utilizing NLP for refinement.
Additionally, handling special characters or disruptive HTML tags would enhance data integrity.
  
---

### Conclusion 🌟
This script offers an efficient method for scraping articles from "andhrajyothy.com" across various categories, dynamically saving the data in JSON files. The challenges faced during the development were overcome using a combination of best practices in web scraping, retry mechanisms, and data cleaning. Designed for scalability, the tool can be easily adapted for other web scraping projects. Currently, it scrapes all available pages within each category, ensuring comprehensive content coverage, but it also allows for modifications to target specific page numbers, balancing computational costs and processing time.




