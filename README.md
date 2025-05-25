import threading
import requests
from queue import Queue
import time

class WebCrawler:
    def __init__(self, urls, max_threads=5):
        

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
import pygame
import random
import sys

# 初始化
pygame.init()
WIDTH, HEIGHT = 800, 600
screen = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption("2D 冒险游戏")
clock = pygame.time.Clock()

# 颜色
WHITE = (255, 255, 255)
RED = (255, 0, 0)
BLACK = (0, 0, 0)

# 玩家类
class Player(pygame.sprite.Sprite):
    def __init__(self):
        super().__init__()
        self.image = pygame.Surface((40, 60))
        self.image.fill((0, 100, 255))
        self.rect = self.image.get_rect()
        self.rect.center = (WIDTH // 2, HEIGHT // 2)
        self.health = 100
        self.speed = 5

    def update(self, keys):
        if keys[pygame.K_a]: self.rect.x -= self.speed
        if keys[pygame.K_d]: self.rect.x += self.speed
        if keys[pygame.K_w]: self.rect.y -= self.speed
        if keys[pygame.K_s]: self.rect.y += self.speed

        # 边界限制
        self.rect.clamp_ip(screen.get_rect())

# 怪物类
class Enemy(pygame.sprite.Sprite):
    def __init__(self):
        super().__init__()
        self.image = pygame.Surface((30, 30))
        self.image.fill((200, 0, 0))
        self.rect = self.image.get_rect()
        self.rect.x = random.randint(0, WIDTH - 30)
        self.rect.y = random.randint(0, HEIGHT - 30)
        self.speed = 2

    def update(self, player_rect):
        if self.rect.x < player_rect.x: self.rect.x += self.speed
        if self.rect.x > player_rect.x: self.rect.x -= self.speed
        if self.rect.y < player_rect.y: self.rect.y += self.speed
        if self.rect.y > player_rect.y: self.rect.y -= self.speed

# 物品类
class Item(pygame.sprite.Sprite):
    def __init__(self, x, y):
        super().__init__()
        self.image = pygame.Surface((20, 20))
        self.image.fill((0, 255, 0))
        self.rect = self.image.get_rect(topleft=(x, y))

# 创建精灵组
player = Player()
player_group = pygame.sprite.Group(player)

enemy_group = pygame.sprite.Group()
for _ in range(5):
    enemy_group.add(Enemy())

item_group = pygame.sprite.Group()
for _ in range(3):
    x = random.randint(0, WIDTH - 20)
    y = random.randint(0, HEIGHT - 20)
    item_group.add(Item(x, y))

# 游戏主循环
def game_loop():
    running = True
    font = pygame.font.SysFont(None, 36)
    collected = 0

    while running:
        clock.tick(60)
        screen.fill(WHITE)

        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                pygame.quit()
                sys.exit()

        keys = pygame.key.get_pressed()
        player.update(keys)

        # 更新怪物
        for enemy in enemy_group:
            enemy.update(player.rect)

        # 检测碰撞
        if pygame.sprite.spritecollide(player, enemy_group, False):
            player.health -= 1
            if player.health <= 0:
                print("你输了！")
                running = False

        collected_items = pygame.sprite.spritecollide(player, item_group, True)
        collected += len(collected_items)

        # 画出精灵
        item_group.draw(screen)
        player_group.draw(screen)
        enemy_group.draw(screen)

        # 显示血量 & 收集数
        health_text = font.render(f"HP: {player.health}", True, RED)
        screen.blit(health_text, (10, 10))
        item_text = font.render(f"收集物品: {collected}/3", True, BLACK)
        screen.blit(item_text, (10, 50))

        if collected >= 3:
            win_text = font.render("你赢了！", True, (0, 150, 0))
            screen.blit(win_text, (WIDTH // 2 - 50, HEIGHT // 2))

        pygame.display.flip()

# 运行游戏
game_loop()
