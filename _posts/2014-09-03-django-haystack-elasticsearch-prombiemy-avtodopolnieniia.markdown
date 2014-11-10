---
layout: post
title:  "Django Haystack + Elasticsearch: промбемы автодополнения"
date:   2014-09-03 16:39:00
tags: [programming, python, django]
---

В своих проектах часто использую **Django Haystack** для поиска, в качестве бекэнда выступает **Elasticsearch**. К сожалению, Haystack предоставляет довольно мало возможностей для настройки индексов, все они [вшиты в код](https://github.com/toastdriven/django-haystack/blob/master/haystack/backends/elasticsearch_backend.py#L48). Из-за чего стречаются малкие и серьезные проблемы. С одной из таких проблем столкнулся, когда нужно было реализовать поиск с автодополнением (autocomplete).

Обратимся к документации:

{% highlight python %}
import datetime
from haystack import indexes
from myapp.models import Note


class NoteIndex(indexes.SearchIndex, indexes.Indexable):
    text = indexes.CharField(document=True, use_template=True)
    author = indexes.CharField(model_attr='user')
    pub_date = indexes.DateTimeField(model_attr='pub_date')
    # We add this for autocomplete.
    content_auto = indexes.EdgeNgramField(model_attr='content')

    def get_model(self):
        return Note

    def index_queryset(self, using=None):
        """Used when the entire index for model is updated."""
        return Note.objects.filter(pub_date__lte=datetime.datetime.now())
{% endhighlight %}

{% highlight python %}
from haystack.query import SearchQuerySet

SearchQuerySet().autocomplete(content_auto='old')
{% endhighlight %}

Сказано, что при поиске строки **old**, мы получим слудуюшие результаты:

`goldfish, cuckold и older`

Все хорошо, пока мы не включили в поиск слово длиннее трех символов. При поиске строки **gold**, результаты будут неожиданными:

`goldfish, cuckold, older, golrang, golf`

То есть в поисковую выдачу попадут слова включающе любые три символа в поисковом запросе. Причина этому дефольные настройки Elasticsearch, которые [прадлагает](https://github.com/toastdriven/django-haystack/blob/master/haystack/backends/elasticsearch_backend.py#L709) Django Haystack. Как видим, `edge_ngram` имеет стандартный анализатор. Но нам нужно, чтобы поиск находил, только точные совпадения и отсекал лишнее. Чтобы решить эту задачу, мы должны в маппинге явно указать, `index_analyzer` и `search_analyzer`. Это можно сделать явно, запросом к ES:

{% highlight bash %}
PUT /my_index/my_type/_mapping
{
    "my_type": {
        "properties": {
            "content_auto": {
                "index_analyzer":  "autocomplete",
                "search_analyzer": "standard"
            }
        }
    }
}
{% endhighlight %}

Теперь можно убедиться, что все работает правильно и в результатах нет мусора.

К сожалению, при перестроении индекса, все опять сломается. Чтобы этого не произошло придется написать свою версию `ElasticsearchSearchBackend`, указав свои настройки для `edge_ngram` в `FIELD_MAPPINGS`.

{% highlight python %}
from haystack.backends.elasticsearch_backend import ElasticsearchSearchBackend, ElasticsearchSearchEngine, \
    FIELD_MAPPINGS, DEFAULT_FIELD_MAPPING
from haystack.constants import DJANGO_CT, DJANGO_ID


FIELD_MAPPINGS['edge_ngram'] = {'type': 'string', 'index_analyzer': 'edgengram_analyzer', 'search_analyzer': 'standard'}


class ElasticsearchCustomBackend(ElasticsearchSearchBackend):
    DEFAULT_SETTINGS = {
        'settings': {
            "analysis": {
                "analyzer": {
                    "ngram_analyzer": {
                        "type": "custom",
                        "tokenizer": "lowercase",
                        "filter": ["haystack_ngram"]
                    },
                    "edgengram_analyzer": {
                        "type": "custom",
                        "tokenizer": "lowercase",
                        "filter": ["haystack_edgengram"]
                    }
                },
                "tokenizer": {
                    "haystack_ngram_tokenizer": {
                        "type": "nGram",
                        "min_gram": 3,
                        "max_gram": 15,
                    },
                    "haystack_edgengram_tokenizer": {
                        "type": "edgeNGram",
                        "min_gram": 2,
                        "max_gram": 15,
                        "side": "front"
                    }
                },
                "filter": {
                    "haystack_ngram": {
                        "type": "nGram",
                        "min_gram": 3,
                        "max_gram": 15
                    },
                    "haystack_edgengram": {
                        "type": "edgeNGram",
                        "min_gram": 2,
                        "max_gram": 15
                    }
                }
            }
        }
    }

    def build_schema(self, fields):
        content_field_name = ''
        mapping = {
            DJANGO_CT: {'type': 'string', 'index': 'not_analyzed', 'include_in_all': False},
            DJANGO_ID: {'type': 'string', 'index': 'not_analyzed', 'include_in_all': False},
        }

        for field_name, field_class in fields.items():
            field_mapping = FIELD_MAPPINGS.get(field_class.field_type, DEFAULT_FIELD_MAPPING).copy()
            if field_class.boost != 1.0:
                field_mapping['boost'] = field_class.boost

            if field_class.document is True:
                content_field_name = field_class.index_fieldname

            # Do this last to override `text` fields.
            if field_mapping['type'] == 'string':
                if field_class.indexed is False or hasattr(field_class, 'facet_for'):
                    field_mapping['index'] = 'not_analyzed'
                    del field_mapping['analyzer']

            mapping[field_class.index_fieldname] = field_mapping

        return (content_field_name, mapping)


class ElasticsearchSearchCustomEngine(ElasticsearchSearchEngine):
    backend = ElasticsearchCustomBackend

{% endhighlight %}

Осталось добавить наш бекэнд в настройки.
