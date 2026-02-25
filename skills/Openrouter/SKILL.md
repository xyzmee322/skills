---
name: Openrouter Api
description: > 
  Используй этот скилл, чтобы больше узнать об Openrouter API, посмотреть примеры, настройки, структуру JSON
  Инфа как правильно формировать корректные запросы к ИИ.
---

# База
- Со многими задачами в пайплайне зачастую справляются модели gemini от google. Засчёт своей мультимодальности и хорошей цены.
- Не забывай также передавать temperature и reasoning, а также другие важные параметры.
- Важно передавать user_id пользователя в проекте при запросе. 
- Ловить, логировать ошибки которые возвращает модель. Не отправлять их в чат юзерам. Чекать "finish_reason": content_filter и error


# Структура запроса

```
response = requests.post(
    "https://openrouter.ai/api/v1/chat/completions",
    headers={
        "Authorization": f"Bearer {api_key}",
        "Content-Type": "application/json",
    },
    json={
        "model": "google/gemini-2.5-flash-lite",
        "messages": [{"role": "user", "content": "ку"}],
        "reasoning": {"effort": "low"},    # У моделей gemini ризонинг не ограничивается токенами, только через effort.
        "include_reasoning": True,    # Если важно логировать размышления или показывать их юзеру.
        "temperature": 0.3,         # Всегда важно передавать и настраивать.
        "max_tokens": 1000,         # Чисто верхнюю границу указать чтобы без бабок не остатся. Ризнонинг у большинства моделей тоже можно токенами ограничить.
        "stream": False,            # Тру только если нужно показывать ответ кусками - иначе не надо. Тк на каждый кусочек срабатывает Safety фильтр, когда на полный ответ только один раз. 
        "user": str(user_id),       # Передаём user_id обратившегося
        "extra_body": {             # Новые параметры или специфические настройки OpenRouter
            "provider": {
                "order": ["Google"], "allow_fallbacks": False,      # Слать запросы только определённому провайдеру - так не стоит делать без запроса.
                "require_parameters": True       # Отправлять тем провайдерам, которые поддерживают все параметры - может быть полезно.
            }, 
        }            
    }
)
```

# Пример JSON Output

```
{
    "id": "gen-1772036036-03sYeRNNFhHlQjQZpj70",
    "provider": "Google",
    "model": "google/gemini-2.5-flash",
    "object": "chat.completion",
    "created": 1772036042,
    "choices": [
        {
            "logprobs": null,
            "finish_reason": "stop",
            "native_finish_reason": "STOP",
            "index": 0,
            "message": {
                "role": "assistant",
                "content": "Привет! Чем могу помочь?",
                "refusal": null,
                "reasoning": null,
                "annotations": []
            }
        }
    ],
    "usage": {
        "prompt_tokens": 2,
        "completion_tokens": 362,
        "total_tokens": 364,
        "cost": 0.0009056,
        "is_byok": false,
        "prompt_tokens_details": {
            "cached_tokens": 0,
            "cache_write_tokens": 0,
            "audio_tokens": 0,
            "video_tokens": 0
        },
        "cost_details": {
            "upstream_inference_cost": 0.0009056,
            "upstream_inference_prompt_cost": 6e-07,
            "upstream_inference_completions_cost": 0.000905
        },
        "completion_tokens_details": {
            "reasoning_tokens": 356,
            "image_tokens": 0
        }
    }
}
```

- В идеале парсить стоимость запросов и хранить в отдельной таблице бд важные данные. В дальнейшем можно будет легко чекать траты, подтягивать данные для статы
Пример таблицы:  id запроса | модуль пайплайна, где вызывается | Модель | Токены | Стоимость usd | user_id | created_at

Доп метаданные запроса можно достать зная gen id через эндпоинт "https://openrouter.ai/api/v1/generation?id=gen-1772030107-1FJRHH9DliPkfVmjcn64"
Из интересного:

{"data":{"latency":1643,"moderation_latency":null,"generation_time":3881,"provider_responses":[{"endpoint_id":"ebdbef25-737d-4ac3-9e98-fd3928724e45","is_byok":false,"latency":327,"model_permaslug":"google/gemini-2.5-flash-lite","provider_name":"Google","status":429},{"endpoint_id":"ce839073-aa24-4f29-8358-15b319bd05ec","is_byok":false,"latency":1643,"model_permaslug":"google/gemini-2.5-flash-lite","provider_name":"Google AI Studio","status":200}],"api_type":"completions","id":"gen-1772030107-1FJRHH9DliPkfVmjcn64","upstream_id":null,"total_cost":0.0001402,"cache_discount":null,"upstream_inference_cost":0,"provider_name":"Google AI Studio"}}

Видно ретраи, latency, провайдеров.