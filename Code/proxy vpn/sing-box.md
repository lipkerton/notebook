клиент: https://sing-box.sagernet.org/installation/package-manager/
# server shadowtls + shadowsocks
референс: https://bulianglin.com/archives/sing-box.html
серверная конфигурация была проста:
```JSON
{
  "inbounds": [
    {
      "type": "shadowtls",
      "listen_port": 443,
      "handshake": {
        "server": "www.bing.com",
        "server_port": 443 
      },
      "detour": "shadowsocks-in"
    },
    {
      "type": "shadowsocks",
      "tag": "shadowsocks-in",
      "listen": "127.0.0.1",
      "method": "2022-blake3-aes-128-gcm",
      "password": "8JCsPssfgS8tiRwiMlhARg=="
    }
  ]
}
```
однако она стала сложнее, когда для shadowTLS выпустили новые версии.
теперь нужно заполнять пользователей и пароли:
```JSON
{
	"inbounds: : [
		{
			"type": "shadowtls",
			"listen": "::",       // слушаем все IPv6 и IPv4 интерфейсы.
			"listen_port": 443,
			"version": 3,         // версия ShadowTLS.
			"users": [
				{
					"name": "<user_name>",
					"password": "<user_password>"
				}
			],
			"handshake": {              // какой сервер будем использовать для наеба.
				"server": "www.vk.ru",
				"server_port": 443
			},
			"detour": "shadowsocks-in", // ShadowTLS направит данные в Shadowsocks.
			"strinct_mode": "true"      // заставляем ShadowTLS строже относится к процедуре рукопожатия.
		},
		{
			"type": "shadowsocks",
			"tag": "shadowsocks-in",
			"listen": "127.0.0.1",
			"listen_port": 8388,
			"method": "2022-blake3-aes-128-gcm",
			"password": "<пароль>"
		}
	]
}
```
пароль `shadowsocks` должен строго соответствовать методу шифрования, поэтому в `sing-box` утилите есть возможность создавать такие пароли:
```Bash
sing-box generate rand 16 --base64
```
# client shadowtls + shadowsocks
## proxy
теперь нужно настроить крутые штуки на клиенте.
клиенты для MacOS и iOS содержат в себе старые версии `sing-box`, поэтому и синтаксис там будет старый. при этом на сервере я использую новую `1.13` версию `sing-box`. но это не мешает.
клиент можно настроить так, чтобы он решал куда ему перенаправить некоторые запросы, поэтому я сразу на нем определил правила, которые запрещают отправлять запросы к русским доменам через прокси.
так же я описал два DNS - локальный и глобальный, где локальный будет нужен, чтобы определять адреса русских доменов.
```JSON
{
	"dns": {
		"servers": [
			{
				"tag": "dns-remote",
				"address": "tls://8.8.8.8"
			},
			{
				"tag": "dns-local",
				"address": "77.88.8.8",
				"detour": "direct"
			}
		],
		"rules": [
			{
				"domain_suffix": [
					".ru"
				],
				"server": "dns-local"
			}
		]
	},
  "inbounds": [
    {
      "type": "mixed",
      "listen_port": 1080,
      "sniff": true,
      "set_system_proxy": true
    }
  ],
  "outbounds": [
    {
      "type": "shadowsocks",
      "method": "2022-blake3-aes-128-gcm",
      "password": "MiRMvoYWnzV7KVLTHZrk9A==",
      "detour": "shadowtls-out",
      "multiplex": {
        "enabled": true
      }
    },
    {
      "type": "shadowtls",
      "tag": "shadowtls-out",
      "server": "test",
      "server_port": 443,
      "version": 3,
      "password": "veryveryverystrong",
      "tls": {
        "enabled": true,
        "server_name": "www.vk.ru",
        "utls": {
          "enabled": true,
          "fingerprint": "chrome"
        }
      }
    },
    {
      "type": "direct",
      "tag": "direct"
    },
    {
      "type": "block",
      "tag": "block"
    }
  ],
  "route": {
    "rules": [
      {
        "domain_suffix": [
          ".ru"
        ],
        "outbound": "direct"
      }
    ]
  }
}
```
это максимально простая конфигурация и она написана специально для MacOS. она *не* работает на iOS и я не проверял работает ли на Windows (должна работать).
## tun
самое интересное - режим туннелирования.
и я пока не смог его настроить должным образом.
