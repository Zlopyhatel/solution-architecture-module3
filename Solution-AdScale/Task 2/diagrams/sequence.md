@startuml
skinparam ParticipantPadding 10
skinparam BoxPadding 10

participant "DSP-партнёр" as DSP
participant "API Gateway" as GW

box "Bidding Service Domain" #LightBlue
    participant "API Handler" as Handler
    participant "Targeting Filter Engine" as Filter
    participant "Auction Engine" as Auction
    participant "Cache Manager" as Cache
end box

database "Кэш Redis" as Redis
queue "Шина данных Kafka" as Kafka

== Горячий контур (Hot Path < 80 ms) ==

DSP -> GW : HTTP POST /bid (OpenRTB request)
activate GW

GW -> GW : Валидация API-key в памяти & Rate Limiting
GW -> Handler : gRPC bidRequest()
activate Handler

Handler -> Filter : filterCandidates(city, interests)
activate Filter

Filter -> Cache : getIndexes(city, interests)
activate Cache

Cache -> Redis : SMEMBERS target:city / target:interest
activate Redis
Redis --> Cache : Список ID подходящих кампаний
deactivate Redis

Cache --> Filter : campaign_ids
deactivate Cache

Filter -> Auction : runAuction(campaign_ids)
activate Auction

Auction -> Cache : getCampaignDetails(campaign_ids)
activate Cache
Cache -> Redis : HMGET campaign:ID (ставки, лимиты, баннеры)
activate Redis
Redis --> Cache : Данные кампаний
deactivate Redis
Cache --> Auction : campaign_details
deactivate Cache

Auction -> Auction : Расчет победителя аукциона в памяти

Auction --> Filter : auction_result (победитель, ставка, баннер)
deactivate Auction

Filter --> Handler : bid_response
deactivate Filter

Handler --> GW : gRPC bidResponse(price, banner_url)
deactivate Handler

GW --> DSP : HTTP 200 OK (OpenRTB response)
deactivate GW

== Асинхронный хвост (Фоновый процесс) ==

Auction -> Cache : logImpression(bid_id, price, advertiser_id)
activate Auction
activate Cache
Cache -> Kafka : pushEvent(AdImpressionLogged)
activate Kafka
Kafka --> Cache : ACK
deactivate Kafka
Cache --> Auction : success
deactivate Cache
deactivate Auction

@endum
