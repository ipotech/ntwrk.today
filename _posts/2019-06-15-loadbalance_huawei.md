---
layout: post
title:  "Балансировка трафика на оборудовании Huawei в MPLS сетях и не только"
tags: loadbalance mpls huawei
author: "riddler63"
---

Варианты балансировки трафика на оборудовании Huawei.

## Типы балансировки в MPLS VPN

1. Балансировка трафика внутри MPLS L2VPN (IP over Ethernet over MPLS)
1. Балансировка трафика внутри MPLS L3VPN  (IP over MPLS)
1. Балансировка трафика L2VPN/L3VPN внутри L2VPN
1. Балансировка трафика других типов туннелей внутри L2VPN/L3VPN
   * GRE
   * PPTP
   * GTP
   * PPPoE

Всегда остаётся возможность наличия других вариантов балансировки, не упомянутых в списке выше.

## Типы узлов

1. Ingress PE
1. Egress PE
1. Transit P
1. L3 (IP) forwarding
1. L2 (ethernet) forwarding

## Основные проблемы

1. Большинство оборудования при балансировке по умолчанию использует MPLS label stack для вычисления hash вследствие отсутствия разнообразия в MPLS стеке меток в L2VPN/L3VPN; 
   * При использовании `apply-label per-instance` в L3VPN в общем случае будет 2 одинаковых метки в MPLS стеке;
   * L2VPN также будет иметь 2 одинаковых ветки;
1. Трафик завёрнут в тот или иной туннель (GRE/PPTP/GTP/PPPOE) внутри MPLS L3VPN/L2VPN;
1. Балансировка L2VPN/L3VPN внутри L2VPN;
1. Поляризация трафика. Заключатеся в том, что однотипный трафик (у которого совпадают `sIP/dIP` (eNodeB X.X.X.[всегда нечетный] <--> RNC Y.Y.Y.любой)) всегда форвардится по одному и тому же аплинку узла при наличии как минимум двух. Как правило встречается только на маломощных и старых платформах, в которых используется примитивная hash функция (XOR) без каких-либо случайных hash seed. Проблема характерна для MBH сетей;
1. Полное отсутствие разнообразия в payload L2VPN/L3VPN (не решается);
   * Одинаковые sMAC/dMAC/VLAN/L2-protocol + одинаковые L3/L4 заголовки в L2VPN;
   * Одинаковые L3/L4 заголовки в L3VPN.

Возможности оборудования по балансировке трафика в той или иной роли сильно различаются, в рамках одной платформы существуют платы с разными способностями "заглянуть" внутрь трафика и посчитать hash для балансировки трафика по ECMP/LAG/Tunnel.

Балансировка трафика на платформах с N линейных плат, как правило, делается на Ingress LPU, поэтому важно смотреть, откуда приходит небалансируемый трафик, а не куда он уходит.

## Service Router NE40-X3/X8/X16 (A)

Оборудование Huawei NE40 использует различную логику и конфигурационные команды, которые её могут изменить, в зависимости от того, чем это оборудование является для трафика/сервиса (см. [типы узлов](#Типы%20узлов)).

Приведенные ниже команды работают для ECMP и LAG одновременно, где это возможно.

Основные команды:

1. `load-balance hash-fields ip` - балансировка чистого IP трафика (GRT, VRF-lite) в роли обычного маршрутизатора и Ingress PE/Egress PE для MPLS L3VPN. По умолчанию L4 (5 tuple);
1. `load-balance hash-fields mac` - балансировка чистого L2 трафика (vlan, bridge-domain) в роли обычного коммутатора и Ingress PE/Egress PE для MPLS VPLS. По умолчанию L4 (5 tuple);
1. `load-balance hash-fields vll` - балансировка чистого L2 трафика в роли Ingress PE/Egress PE для MPLS VLL. По умолчанию L4 (5 tuple);
1. `load-balance hash-fields mpls` - балансировка MPLS трафика в роли Transit P. По умолчанию по IP. Конфигурация `payload-header` позволит решить существуенную часть проблем балансировки на транзитном узле (см документацию). При наличии после MPLS заголовка пакета IP, используется 5 tuple заголовка IP данного пакета, даже если это L2VPN, внутри которого идёт трафик IP. Если после MPLS заголовка нет пакета IP, то используется MPLS метка на дне MPLS Label stack и первые 12 байт MPLS payload.
1. `load-balance identify pppoe` - балансировка PPPoE трафика по IP 5 tuple внутри PPPoE. Актуально для BNG;
1. `nat load-balance hash-fields` - балансировка NAT трафика после прохождения через сервисную плату (VSUI/VSUF). По умолчанию L3;
1. `load-balance hash-arithmetic` - изменение алгоритма рассчёта hash для балансировки трафика. Настраивается отдельно на каждый слот.

Экзотика:

1. `load-balance hash-fields ip l3 (Interface view)` - балансировка L3VPN/L2VPN over L2VPN и IPoIP сценариев. Настраивается на интерфейсе и имеет ряд серьезных ограничений по функциональности после конфигурации. Заглядывает так далеко в кроличью нору, что теряет ряд способностей;
1. `load-balance hash-fields l2vxlan-vni enable` `load-balance hash-fields l3vxlan-vni enable` - балансировка VXLAN;
1. `load-balance hash-seed` - решение проблемы поляризации на старых платах. Использует случайный seed как аргумент при подсчёте hash.

## Коммутаторы Huawei

Балансировка в коммутаторах сильно зависит от чипа и версии софта.

Достаточное количество проблем с балансировкой можно решить созданием `load-balance-profile` и конфигурацией его на LAG. С его помощью можно определить поля, по которым будет производиться балансировка для следуюших типов трафика:

1. IPv4
1. IPv6
1. L2
1. MPLS

Значения по умолчанию далеки от идеальных, желательно решать проблему балансировки по мере её возникновения и включать обработку полей по факту.

Также данный `load-balance-profile` будет по умолчанию использоваться на Stack линках в случае использования обычных портов для формирования логического коммутатора из нескольких физических.

Не ожидайте от коммутаторов поддержки FAT/EL и форвардинга трафика по ECMP через несколько MPLS LDP LSP, такую функциональность необходимо проверять отдельно по документции и спецификации.

### Правильные способы балансировать MPLS трафик

Использование хитрых и ресурсоемких возможностей оборудования для балансировки однотипного трафика не всегда оправдано, при возможности желательно вносить хаос в MPLS label stack. На данный момент существует два RFC, позволяющих это реализовать:

1. [Flow-Aware Transport of Pseudowires over an MPLS Packet Switched Network](https://tools.ietf.org/html/rfc6391)<sup id="a1">[1](#f1)</sup> - на NE40E реализовано для Martini L2VPN (VLL/VPLS);
   * Плюсы
      * Поддержка необходима только на Ingress PE и/или Egress PE;
      * Можно включать по-сервисно;
      * Добавляет всего одну метку в MPLS label stack.
   * Минусы
      * Не может быть использован для L3VPN/EVPN;
1. [The Use of Entropy Labels in MPLS Forwarding](https://tools.ietf.org/html/rfc6790)<sup id="a2">[2](#f2)</sup>;
   * Плюсы
      * Подходит для всех типов сервисов L2VPN/L3VPN/EVPN и так далее;
   * Минусы
      * По разным данным нужна поддержка на всём пути прохождения LSP;
      * Добавляет 2 метки в MPLS label stack;
      * Мало кем реализовано.

В зависимости от софта и платформы возможно настраивать какие поля будут источниками для рассчёта hash для генерации Flow Aware/Entropy Label.

## Балансировка в не-MPLS туннелях

Балансировка инкапсулированного в GRE/GTP/PPTP трафика внутри L3VPN/L2VPN происходит автоматически по:

1. GRE: GRE Key
1. GTP: GTP sequence number
1. PPTP: Enhanced GRE Key

## UCMP

Для использования UCMP необходимо сначала сделать ECMP по IGP cost для интерфейсов/туннелей, в которые будет балансироваться трафик. После этого оборудование может форвардить трафик согласно скорости интерфейсов/туннелей.

Например, если есть следующие интерфейсы LAG 2x10G + 10G physical интерфейс + LAG 5x1G с одинаковым IGP cost = 10, то трафик поделится в пропорциях 4:2:1.

> **Примечание:** Все тестируют, но мало кто решается использовать.

## LDP over TE

Если трафик не балансируется (платформа не может "глубоко" разобрать payload), но есть большая необходимость, то можно создать N MPLS TE (RSVP-TE) туннелей на платформе, которая сможет обработать payload и распределить по ECMP трафик в них, таким образом добавив третью метку в MPLS label stack и создать хоть какое-то разнообразие в стеке меток.  

Переход на более скоростную технологию передачи данных (1G -> 2.5G -> 5G -> 10G -> 25G -> 40G -> 50G -> 100G -> 200G -> 400G...) почти всегда лучше, чем попытки реализовать балансировку на сложной логике.

## Ссылки
<b id="f1">1</b>. RFC 6391: [https://tools.ietf.org/html/rfc6391](https://tools.ietf.org/html/rfc6391) [↩](#a1)<br/>
<b id="f2">2</b>. RFC 6790: [https://tools.ietf.org/html/rfc6790](https://tools.ietf.org/html/rfc6790) [↩](#a2)<br/>