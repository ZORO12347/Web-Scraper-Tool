# README

## Web Scraping Script for Andhrajyothy.com

### Overview
This Python script scrapes articles from the website "andhrajyothy.com" across various categories and saves them into individual JSON files as soon as a category is completed. The script uses the BeautifulSoup library for HTML parsing, the requests library for making HTTP requests, and the random library to mimic different user agents and avoid being blocked.

### Requirements
- Python 3.7 or higher
- Libraries:
  - requests
  - beautifulsoup4
  - json

### Creating a Virtual Environment
1. Create a virtual environment named `myenv`:
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

### Script Workflow
1. Loads previously scraped hrefs to avoid re-scraping.
2. For each category, scrapes article links from all available pages.
3. Processes each article to extract the title, URL, category, and content.
4. Saves the articles into a JSON file named `scraped_articles_<category>.json` as soon as a category is completed.
5. Updates the set of scraped hrefs to avoid duplicating work in future runs.

### Example Output
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

### Tags and Attributes
- `<a>` tag and `href` attribute:
  - The `<a>` tag defines a hyperlink.
  - The `href` attribute indicates the link’s destination.
  - Example:
    ```html
    <a href="https://www.example.com">This is a link</a>
    ```
- `<p>` tag:
  - The `<p>` tag defines a paragraph.
  - Example:
    ```html
    <p>This is a paragraph.</p>
    ```

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

### Challenges and Solutions
1. **Content Extraction Issues**:
   - **Problem**: The code was running but no content was being extracted for some articles.
   - **Solution**: Implemented a fallback mechanism to try an alternative method for extracting content if the primary selector fails.

2. **Handling Transient Errors**:
   - **Problem**: The script encountered occasional transient errors such as network issues or server overloads.
   - **Solution**: Added a retry mechanism with exponential backoff to handle transient errors gracefully.

3. **Dynamic Content Loading**:
   - **Problem**: Some pages load content dynamically using JavaScript, which BeautifulSoup cannot handle.
   - **Solution**: Used requests and retry strategy to fetch the static content and relied on alternative selectors to extract the required information.

### Insights
- **Importance of Clean Data**: Cleaning the scraped data (removing unnecessary characters,

 handling missing content, etc.) ensured that the content was usable for further analysis or processing.
- **Dynamic Saving**: Saving JSON files dynamically as soon as a category is completed not only helped in avoiding data loss in case of interruptions but also allowed for easier debugging and validation of the scraping process.

### Conclusion
This script provides a reliable and efficient way to scrape articles from "andhrajyothy.com" across multiple categories, saving them into JSON files dynamically. The challenges faced during the development were overcome using a combination of best practices in web scraping, retry mechanisms, and data cleaning. This tool can be extended or adapted for similar web scraping projects, demonstrating the power of Python for data extraction and automation tasks. Currently, the script is designed to scrape all available pages in each category to ensure comprehensive coverage. However, the script can be modified to scrape a specific number of pages if needed, providing flexibility in managing computational expense and processing time.
