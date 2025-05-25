import threading
import requests
from queue import Queue
import time

class WebCrawler:
    def __init__(self, urls, max_threads=5):
        self.urls = urls
        self.max_threads = max_threads
        self.queue = Queue()
        self.results = {}
        self.lock = threading.Lock()
        self.total = len(urls)
        self.completed = 0

    def fetch(self):
        while True:
            url = self.queue.get()
            if url is None:
                break
            try:
                response = requests.get(url, timeout=5)
                content = response.text
            except Exception as e:
                content = f"Error: {e}"
            with self.lock:
                self.results[url] = content
                self.completed += 1
                print(f"Fetched {self.completed}/{self.total}: {url}")
            self.queue.task_done()

    def run(self):
        # 填充任务队列
        for url in self.urls:
            self.queue.put(url)

        threads = []
        for _ in range(self.max_threads):
            t = threading.Thread(target=self.fetch)
            t.start()
            threads.append(t)

        self.queue.join()

        # 停止线程
        for _ in range(self.max_threads):
            self.queue.put(None)
        for t in threads:
            t.join()

        print("All tasks completed.")

if __name__ == "__main__":
    urls = [
        "https://www.python.org",
        "https://www.google.com",
        "https://www.github.com",
        "https://nonexistent.baddomain",
        # 你可以加更多网址测试
    ]
    crawler = WebCrawler(urls, max_threads=3)
    crawler.run()
    # 结果存在 crawler.results 字典里，key是url，value是内容或错误信息
import requests
from bs4 import BeautifulSoup
import pandas as pd
import time

def fetch_douban_top250(page=0):
    url = f"https://movie.douban.com/top250?start={page*25}&filter="
    headers = {
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko)"
                      " Chrome/90.0.4430.212 Safari/537.36"
    }
    response = requests.get(url, headers=headers)
    response.raise_for_status()
    return response.text

def parse_movie_info(html):
    soup = BeautifulSoup(html, 'html.parser')
    items = soup.select('div.item')
    movies = []
    for item in items:
        title = item.select_one('span.title').get_text(strip=True)
        rating = float(item.select_one('span.rating_num').get_text(strip=True))
        people = item.select_one('div.star > span:nth-child(4)').get_text(strip=True)
        # 评价人数格式：'123456人评价'，提取数字
        people_count = int(people.replace('人评价', '').replace(',', ''))
        movies.append({
            'title': title,
            'rating': rating,
            'people_count': people_count
        })
    return movies

def main():
    all_movies = []
    for page in range(10):  # 爬取前10页，共250部电影
        print(f"Fetching page {page + 1}...")
        html = fetch_douban_top250(page)
        movies = parse_movie_info(html)
        all_movies.extend(movies)
        time.sleep(1)  # 避免请求过快被封

    df = pd.DataFrame(all_movies)
    print("数据爬取完成，开始分析")

    # 简单分析
    print("\n电影数量：", len(df))
    print("平均评分：", df['rating'].mean())
    print("评分分布：")
    print(df['rating'].value_counts().sort_index())

    # 按评分分组，计算评价人数均值
    group = df.groupby('rating')['people_count'].mean()
    print("\n不同评分对应的平均评价人数：")
    print(group)

    # 保存为csv文件
    df.to_csv("douban_top250.csv", index=False)
    print("\n数据已保存到 douban_top250.csv")

if __name__ == "__main__":
    main()
