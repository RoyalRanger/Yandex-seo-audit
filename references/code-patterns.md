# Готовые паттерны кода (FastAPI + Jinja2)

Взято дословно из внедрения на spvrwa.ru (`core.py`, `templates/base.html`,
`templates/article.html`, `templates/index.html`, `routers/news.py`). Адаптируй
имена таблиц/полей под конкретный сайт — сама структура переносима.

## Canonical URL с вайтлистом query-параметров

`core.py`:

```python
# Query-параметры, которые представляют реально другой контент (заслуживают
# свой canonical). Всё остальное — трекинговые параметры (_ym_debug, utm_*,
# случайный мусор) — отбрасывается, чтобы такие варианты не плодили дубли
# в индексе (реальный кейс: /?_ym_debug=2 сам утёк и получил индексацию
# как отдельный URL).
CANONICAL_PARAM_WHITELIST = {"category", "page"}


def canonical_url(request) -> str:
    kept = [(k, v) for k, v in request.query_params.multi_items() if k in CANONICAL_PARAM_WHITELIST]
    base = f"{request.url.scheme}://{request.url.netloc}{request.url.path}"
    if not kept:
        return base
    query = "&".join(f"{k}={v}" for k, v in kept)
    return f"{base}?{query}"


templates.env.globals["canonical_url"] = canonical_url
```

`base.html` (в `<head>`, оборачивай в overridable block, если у страниц бывают
собственные canonical, например пагинация):

```html
<link rel="canonical" href="{% block canonical %}{{ canonical_url(request) }}{% endblock %}">
<meta property="og:site_name" content="ИМЯ САЙТА">
<meta property="og:type" content="{% block og_type %}website{% endblock %}">
<meta property="og:title" content="{% block og_title %}{{ self.title() }}{% endblock %}">
<meta property="og:description" content="{% block og_description %}{{ self.description() }}{% endblock %}">
<meta property="og:url" content="{{ canonical_url(request) }}">
{% block og_image %}{% endblock %}
```

## JSON-LD для статьи (NewsArticle)

`article.html`:

```html
{% block og_type %}article{% endblock %}
{% block og_image %}{% if article['image_url'] %}<meta property="og:image" content="{{ request.url.scheme }}://{{ request.url.netloc }}{{ article['image_url'] }}">{% endif %}{% endblock %}

<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "NewsArticle",
  "headline": {{ article['title']|tojson }},
  "description": {{ article['summary']|tojson }},
  "datePublished": {{ article['published_at']|tojson }},
  {% if article['image_url'] %}"image": [{{ (request.url.scheme + '://' + request.url.netloc + article['image_url'])|tojson }}],{% endif %}
  "author": {"@type": "Organization", "name": {{ (article['source_name'] or 'ИМЯ САЙТА')|tojson }}},
  "publisher": {
    "@type": "Organization",
    "name": "ИМЯ САЙТА",
    "logo": {"@type": "ImageObject", "url": "{{ request.url.scheme }}://{{ request.url.netloc }}/static/apple-touch-icon.png"}
  },
  "mainEntityOfPage": {{ canonical_url(request)|tojson }}
}
</script>
```

Важно: все поля идут через `|tojson`, никогда через ручную конкатенацию строк —
заголовок с кавычками/бэкслешами иначе ломает JSON молча.

## JSON-LD для главной (WebSite + поиск по сайту)

`index.html`, только на настоящей корневой странице (не на отфильтрованных
category/page вариантах — иначе SearchAction дублируется бессмысленно):

```html
{% if not active_category %}
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "WebSite",
  "name": "ИМЯ САЙТА",
  "url": "{{ request.url.scheme }}://{{ request.url.netloc }}/",
  "potentialAction": {
    "@type": "SearchAction",
    "target": "{{ request.url.scheme }}://{{ request.url.netloc }}/search?q={search_term_string}",
    "query-input": "required name=search_term_string"
  }
}
</script>
{% endif %}
```

## favicon.ico на корне

Яндекс-робот ищет `/favicon.ico` на корне независимо от `<link rel="icon">` в
`<head>` и не всегда подхватывает SVG-фавиконку:

```python
FAVICON_ICO = Path(__file__).parent.parent / "static" / "favicon.ico"

@router.get("/favicon.ico")
async def favicon_ico():
    return FileResponse(FAVICON_ICO, media_type="image/x-icon")
```

## robots.txt: блокировка тонких/дублирующих страниц

```python
ROBOTS_TXT = """User-agent: *
Allow: /
Disallow: /admin/
Disallow: /internal/
Disallow: /search

Sitemap: https://ДОМЕН/sitemap.xml
"""

@router.get("/robots.txt", response_class=PlainTextResponse)
async def robots_txt():
    return ROBOTS_TXT
```

`/search` блокируется потому что результаты внутреннего поиска — тонкий,
дублирующий контент (тот же список статей, что и на главной/категориях, просто
отфильтрованный query-строкой); индексация таких URL размывает вес без пользы.

## Верификация после деплоя

```bash
curl -s https://ДОМЕН/ | grep -o '<link rel="canonical"[^>]*>'
curl -s https://ДОМЕН/article/СЛАГ | python3 -c "
import sys, re, json
html = sys.stdin.read()
m = re.search(r'<script type=\"application/ld\+json\">(.*?)</script>', html, re.S)
print(json.loads(m.group(1)))  # упадёт с ошибкой, если JSON невалиден
"
curl -s https://ДОМЕН/robots.txt
```
