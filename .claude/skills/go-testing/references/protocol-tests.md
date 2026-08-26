# Тесты протокольного поведения

Читать, когда тест проверяет поведение, заданное стандартом: разбор и сборку SDP,
SRTP-ключи, RTP-пакеты, обмен INVITE/ACK/BYE, offer/answer.

## Правило

Такой тест **обязан** назвать в комментарии норму, на которой основано ожидание, и само
правило. Не «что делает код», а «по какому стандарту это правильно» — иначе ревьюер не
отличит проверку стандарта от закрепления текущего поведения.

- Ссылка ставится там, где проверяется нетривиальное правило: у `t.Run`-подкейса, у
  конкретной ассерции или в шапке теста.
- Формат: `RFC <num> §<section> — <правило одной строкой>`. Полный URL
  (`https://www.rfc-editor.org/rfc/rfc<num>.html`) — один раз в шапке файла или функции,
  дальше достаточно номера.
- Если кейсов из одного RFC много — общий заголовок-комментарий у функции плюс точечные
  ссылки у спорных ассерций.
- Перед первым использованием нового RFC проверить, что ссылка живая:
  `curl -sI <url> | head -1` (ожидаем `200`).

**Если нормы не нашлось** — это сигнал, что тест фиксирует не стандарт, а текущую
реализацию. Либо найди норму, либо пометь кейс явно: «наше соглашение, не RFC».

## Пример

```go
// buildSDP обязан вернуть тот же транспорт, что прислал вызывающий: RTP/SAVP ↔ RTP/SAVP.
// RFC 3264 §6 — answer повторяет proto предложения; RFC 4568 §5.1 — на SAVP-offer
// с a=crypto answer должен содержать свою crypto-строку, иначе пир шлёт BYE после ACK.
t.Run("SRTP-offer → ответ тоже SAVP с crypto", func(t *testing.T) {
	out, err := buildSDP(offerSDP(true), "198.51.100.2", 10002, "OURKEY999", "9995")
	require.NoError(t, err)
	require.Contains(t, string(out), "RTP/SAVP")
	require.Contains(t, string(out), "inline:OURKEY999")
})

// RFC 3711 §3.2.1 + RFC 4568 §6.1 (AES_CM_128_HMAC_SHA1_80):
// master key 128 бит + master salt 112 бит = 30 байт, передаётся base64 в inline:.
require.Len(t, raw, 30)
```

## Памятка по RFC (телефония и WebRTC)

| Тема | RFC | URL |
|---|---|---|
| SIP: INVITE/180/200/ACK/BYE/CANCEL, диалоги | 3261 | https://www.rfc-editor.org/rfc/rfc3261.html |
| SIP: `rport` — NAT-порт во Via | 3581 | https://www.rfc-editor.org/rfc/rfc3581.html |
| SDP: формат | 4566 | https://www.rfc-editor.org/rfc/rfc4566.html |
| SDP: offer/answer, совпадение proto и кодеков | 3264 | https://www.rfc-editor.org/rfc/rfc3264.html |
| SRTP: ключ 16 байт + соль 14 = 30 | 3711 | https://www.rfc-editor.org/rfc/rfc3711.html |
| SDP security descriptions: `a=crypto:`, `inline:` | 4568 | https://www.rfc-editor.org/rfc/rfc4568.html |
| RTP: payload type, SSRC, тайминг | 3550 | https://www.rfc-editor.org/rfc/rfc3550.html |
| RTP/AVP профиль, статические PT (PCMU 0, PCMA 8) | 3551 | https://www.rfc-editor.org/rfc/rfc3551.html |
| DTMF telephone-event (PT 101, `0-16`) | 4733 | https://www.rfc-editor.org/rfc/rfc4733.html |
| WebRTC: SDP для DTLS-SRTP | 5763 | https://www.rfc-editor.org/rfc/rfc5763.html |
| WebRTC: DTLS-SRTP key transport | 5764 | https://www.rfc-editor.org/rfc/rfc5764.html |

Диагностика тех же протоколов вживую — в `/telephony:sip-diagnostics`, работа с RTP-потоком —
в `/telephony:realtime-audio`.
