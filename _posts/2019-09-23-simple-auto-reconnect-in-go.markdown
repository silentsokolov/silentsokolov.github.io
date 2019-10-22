---
title:  "Простое переподключение к БД в Go"
meta_description: "Пример реализации автоматического переподключения к базе данных в golang"
date:   2019-09-23 11:05:34
---

При разработке web-сервисов, которые должны работать в режиме 24/7, в не последнюю очередь важно правильно работать с ресурсами лежащими за рамками кода, такие как обращение к другим сервисам или работа с базой данных. О последних мы и поговорим.

Не стоит забывать, что разрыв соединение с базой может произойти в любой момент и по многим причинам включая разрыв интернет соединения между серверами, перезагрузка самой БД и тд. Если не уделять этому внимание при разработке, то в лучшем случае ваше приложение упадет и будет перезапущено (если автозапуск настроен). Но лучше перестраховаться заранее.

Ниже приведет пример как это можно сделать простой реконнет к БД работая с Go. Мы используем [pgx](https://github.com/jackc/pgx) реализацию Go клиента для Postgres, которая предоставляет различные плюшки.

```go
package main

import (
	"context"
	"fmt"
	"log"
	"time"

	"github.com/jackc/pgx"
)

type databaseInstance struct {
	conn               *pgx.Conn
	connConfig         pgx.ConnConfig
	maxConnectAttempts int
}

func NewConn(URI string, maxAttempts int) (*databaseInstance, error) {
	connConfig, err := pgx.ParseURI(URI)
	if err != nil {
		return nil, err
	}

	instance := databaseInstance{}
	instance.connConfig = connConfig
	instance.maxConnectAttempts = maxAttempts

	return &instance, err
}

func (db *databaseInstance) reconnect() (*pgx.Conn, error) {
	conn, err := pgx.Connect(db.connConfig)
	if err != nil {
		return nil, fmt.Errorf("Unable to connection to database: %v", err)
	}

	if err = conn.Ping(context.Background()); err != nil {
		return nil, fmt.Errorf("Couldn't ping postgre database: %v", err)
	}

	return conn, err
}

func (db *databaseInstance) GetConn() *pgx.Conn {
	var err error

	if db.conn == nil {
		if db.conn, err = db.reconnect(); err != nil {
			log.Fatalf("%s", err)
		}
	}

	if err = db.conn.Ping(context.Background()); err != nil {
		attempt := 0
		ticker := time.NewTicker(5 * time.Second)
		defer ticker.Stop()

		for {
			select {
			case <-ticker.C:
				if attempt >= db.maxConnectAttempts {
					log.Fatalf("Connection failed after %d attempt\n", attempt)
				}
				attempt += 1
				log.Println("Reconnecting...")
				db.conn, err = db.reconnect()
				if err == nil {
					return db.conn
				}
				log.Printf("Connection was lost. Error: %s. Waiting for 5 sec...\n", err)
			}
		}
	}

	return db.conn
}
```

Код довольно прост: мы перед взятием соединения пингуем БД, если этого не удалось сделать, то джем 5 секунд и пытаемся снова.

Подобное **решение подойдет не всем!** Так как перед каждым запросом осуществляется опрос БД, что создает огромный оверхед и подобное решение лучше использовать в сервисах, которые редко ходят с базу. Для сервисов которые постоянно работают с БД лучше использовать пулы соединений. Но даже работая через пулы, код остается полезным, внеся небольшие изменения можно использовать его при первоначальном соединении с базой, это особенно полезно в работе с docker-compose (старых версий), где сервис мог запуститься раньше базы.
