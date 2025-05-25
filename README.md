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
