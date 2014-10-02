# Cron Scheduler
This module allows you to schedule an event using a Cron specification. That event can be be
scheduled for a single execution or repeated periodically. It can either send or publish a message to the address of
your choice. If the handler of that message returns a response, you can specify where to send that response when
the handler completes.

## Name
The name of the module is vertx-cron.

## Vertx Dependency

com.diabolicallab:vertx-cron:1.1.0

## Configuration

    {
      "address_base": <string>,
      "timezone_name": <string>
    }
    
The Cron Scheduler will use the string specified as the `<address_base>` for registering the handlers your
sender will interact with as well as some handlers used internally. 


It will create public handlers for:
    
    <address_base>.schedule -- used to schedule an event
    <address_base>.cancel -- used to cancel a scheduled event
    
It will also create private handlers for:
    
    <address_base>.map.add -- to add the scheduler id and current timer id.
    <address_base>.map.remove -- to remove an unused scheduler id.
    
These private handlers allow the Cron Scheduler to transport shared information to Cron Scheduler verticals
running on separate nodes.
 
If a `<timezone_name>` is specified, the `<cron_expression>` (mentioned below) will be interpreted with reference to that timezone. If 
none is specified, the timezone of the machine your vertical is running on will be used. It is important to set the 
timezone if you anticipate that a vertical using this module may run in multiple timezones. It is also useful if
you want to schedule an event relative to a particular timezone and don't want to calculate any time offset yourself when
specifying the `<cron_expression>`.

A list of valid timezones is at the end of this document.
 
*Note: At Diabolical Lab, vertical == verticle :-)*

## Configuration Example

    {
      "address_base": "cron.scheduler",
      "timezone_name": "Pacific/Honolulu"
    }

This will cause the Cron Scheduler to create public handlers named: "cron.scheduler.schedule" and "cron.scheduler.cancel"
    


## Schedule an Event

To schedule an event, you need to send a message to this address: `<address_base>`.schedule where `<address_base>`
is the name specified in the configuration. Scheduled events are not persistent. If Vertx restarts, you will have to 
schedule your events again.

The message you send will conform to the following JSON schema:

    {
        "title": "A Cron Scheduler schedule message",
        "type":"object",
        "properties": {
            "cron_expression": {"type": "string"},
            "address": {"type": "string"},
            "message": {"type": "object"},
            "repeat": {"type": "boolean"},
            "action": {"enum": ["send", "publish"]},
            "result_address": {"type": "string"}
        },
        "required": ["cron_expression", "address"]
    }

**cron_expression** is a standard cron expression as is frequently used on Linux systems. As an example, "*/5 * * * * ?" would result in an event
being fired every 5 seconds. 

Most Cron implementation do not allow the scheduling of events down to the second, they will only allow minutely specifications. 
We are borrowing the org.quartz.CronExpression class from the Quartz Scheduler project that *does* allow the specification of seconds. 
Check this documentation [CronExpression] (http://quartz-scheduler.org/api/2.2.0/org/quartz/CronExpression.html) for allowed values.

This [CronMaker Calculator] (http://www.cronmaker.com/) may also help you form the cron_expression.

**address** is the address you want to send a message to on a scheduled basis.

**message** is the message that you want to send to the aforementioned address. It is not required. If it is not specified, the 
Cron Scheduler will send or publish a null message to the address at the time mentioned by the cron_expression.

**repeat** will instruct the Cron Scheduler to either send the message once at the specified time, or repeating it periodically.
It is not required. The default is true.

**action** is the action to take at the specified time. It can be "send" or "publish". If you specify "send" the Cron Scheduler
will send the message to the address and wait for any result. If you specify a result_address, the result of the send will be forwarded to 
that address. If you specify "publish" the message will be published to the address specified and the Cron Scheduler will not wait
to collect any result. The action is not required. If omitted, the default is "send".

**result_address** is the address to which you want any result sent. 

Here is an example schedule message:

    {
        "cron_expression": "0 0 16 1/1 * ? *",
        "address": "stock.quotes.list",
        "message": {
            "ticker": "RHT"
        },
        "repeat": true,
        "action": "send",
        "result_address": "stock.quotes.persist"
    }
  
This message would cause the Cron Scheduler to send the message {"ticker": "RHT"} to "stock.quotes.list" every day (including weekend days) at 16:00. The 
Cron Scheduler would then wait for a response from "stock.quotes.list" and forward the result to "stock.quotes.persist"

In Groovy it would look like this:

    def message = [
      cron_expression: '0 0 16 1/1 * ? *',
      address: "stock.quotes.list",
      message: [ticker: "RHT"],
      repeat: true,
      action: "send",
      result_address: "stock.quotes.persist"
    ]
    vertx.eventBus.send("cron.message.schedule", message) { result ->
      assert result.body.status == 'ok'
    }
    
    //This will be called every day at 16:00
    vertx.eventBus.registerHandler("stock.quotes.list") { message ->
      message.reply([close: 60.5])
    }
    //The Cron Scheduler will forward the reply from "stock.quotes.list" to this handler
    vertx.eventBus.registerHandler("stock.quotes.persist") { message ->
        // message.body == [close: 60.5] 
    }
  
## Scheduler Response

The `<address_base>`.schedule handler will return a message like the following when successful:

    {
        "status":"ok",
        "scheduler_id": "3857544d-cb68-4874-8ed8-82bee3713e3f",
        "next_run_time": "2001-09-11T13:46:30+0000"
    }
    
The `<address_base>`.schedule handler will return a message like the following when unsuccessful:

    {
        "status":"error",
        "message": "cron_expression must be specified"
    }

**status** can be "ok" or "error"

**message** is any informational message

**scheduler_id** is the string identifier of the schedule that the Cron Scheduler created

**next_run_time** is the string representing the time (in ISO format) that the next message will be sent.


## Cancel a Scheduled Event

To cancel a scheduled event you need to send a message to this address: `<address_base>`.cancel where `<address_base>`
is the name specified in the configuration.

The message will be the string returned by the `<address_base>`.schedule in the scheduler_id field.

In Groovy it would look like this:

    vertx.eventBus.send("cron.message.cancel", "3857544d-cb68-4874-8ed8-82bee3713e3f") { result ->
        assert result.body == 'ok'
    }
    
## Cancel Response

The `<address_base>`.cancel handler will return a message like the following when successful:

    {
        "status":"ok"
    }
    
The `<address_base>`.cancel handler will return a message like the following when unsuccessful:

    {
        "status":"error",
        "message": "The scheduler_id was not found"
    }

**status** can be "ok" or "error"

**message** is any informational message

## Additional Examples

## Dynamic message

If you need to send a message periodically, but some of the attributes of that message need to be 
determined at the scheduled time, you can do it like this:

    //this handler will return stock quotes for an exchange by date
    
    vertx.eventBus.registerHandler("stock.quotes.exchange.list") { message ->
      def exchange = message.body.exchange
      def date = message.body.date
      def quotes = StockQuoteService(exchange, date).getQuotes()
      message.reply(quotes)
    }
    
    //this handler will be called by the Cron scheduler.
    
    vertx.eventBus.registerHandler("stock.quotes.exchange.list.request") { message ->
        //the date below will be computed dynamically at the scheduled time
        def request = [
            exchange: 'NYSE',
            date: JsonOutput.toJson(new Date())
        ]
        vertx.eventBus.send("stock.quotes.exchange.list", request) { response ->
            def quotes = response.body
            //now save the quotes
            vertx.eventBus.send("stock.quotes.save", [quotes:quotes])
        }
    }
    
    //now schedule the event for 16:00 each day
    //there is no need to specify the result address since stock.quotes.exchange.list.request
    //will handle the result
    
    def message = [
      cron_expression: '0 0 16 1/1 * ? *',
      address: "stock.quotes.exchange.list.request",
      repeat: true,
    ]
    vertx.eventBus.send("cron.message.schedule", message) { result ->
      assert result.body.status == 'ok'
    }
    
## Publish message

If you wanted to clear a shared map each day at a particular time you can do it like this:

    //this will remove the shared map of the name passed on the message.
    //since in Vertx 2.x shared maps may exist on each vertx instance, we will need to publish 
    //to this address in order to clear all of them
    
    vertx.registerHandler("shared.map.clear") { message ->
        //message.body contains the map name
        vertx.sharedData.removeMap(message.body)
    }
    
    //schedule to publish a message to clear the map named "big.cache" each day at 23:00
    def message = [
      cron_expression: '0 0 23 1/1 * ? *',
      address: "shared.map.clear",
      message: "big.cache",
      repeat: true,
      action: "publish"
    ]
    vertx.eventBus.send("cron.message.schedule", message) { result ->
      assert result.body.status == 'ok'
    }
    
## Valid Timezones
- ACT
- AET
- AGT
- ART
- AST
- Africa/Abidjan
- Africa/Accra
- Africa/Addis_Ababa
- Africa/Algiers
- Africa/Asmara
- Africa/Asmera
- Africa/Bamako
- Africa/Bangui
- Africa/Banjul
- Africa/Bissau
- Africa/Blantyre
- Africa/Brazzaville
- Africa/Bujumbura
- Africa/Cairo
- Africa/Casablanca
- Africa/Ceuta
- Africa/Conakry
- Africa/Dakar
- Africa/Dar_es_Salaam
- Africa/Djibouti
- Africa/Douala
- Africa/El_Aaiun
- Africa/Freetown
- Africa/Gaborone
- Africa/Harare
- Africa/Johannesburg
- Africa/Juba
- Africa/Kampala
- Africa/Khartoum
- Africa/Kigali
- Africa/Kinshasa
- Africa/Lagos
- Africa/Libreville
- Africa/Lome
- Africa/Luanda
- Africa/Lubumbashi
- Africa/Lusaka
- Africa/Malabo
- Africa/Maputo
- Africa/Maseru
- Africa/Mbabane
- Africa/Mogadishu
- Africa/Monrovia
- Africa/Nairobi
- Africa/Ndjamena
- Africa/Niamey
- Africa/Nouakchott
- Africa/Ouagadougou
- Africa/Porto-Novo
- Africa/Sao_Tome
- Africa/Timbuktu
- Africa/Tripoli
- Africa/Tunis
- Africa/Windhoek
- America/Adak
- America/Anchorage
- America/Anguilla
- America/Antigua
- America/Araguaina
- America/Argentina/Buenos_Aires
- America/Argentina/Catamarca
- America/Argentina/ComodRivadavia
- America/Argentina/Cordoba
- America/Argentina/Jujuy
- America/Argentina/La_Rioja
- America/Argentina/Mendoza
- America/Argentina/Rio_Gallegos
- America/Argentina/Salta
- America/Argentina/San_Juan
- America/Argentina/San_Luis
- America/Argentina/Tucuman
- America/Argentina/Ushuaia
- America/Aruba
- America/Asuncion
- America/Atikokan
- America/Atka
- America/Bahia
- America/Bahia_Banderas
- America/Barbados
- America/Belem
- America/Belize
- America/Blanc-Sablon
- America/Boa_Vista
- America/Bogota
- America/Boise
- America/Buenos_Aires
- America/Cambridge_Bay
- America/Campo_Grande
- America/Cancun
- America/Caracas
- America/Catamarca
- America/Cayenne
- America/Cayman
- America/Chicago
- America/Chihuahua
- America/Coral_Harbour
- America/Cordoba
- America/Costa_Rica
- America/Creston
- America/Cuiaba
- America/Curacao
- America/Danmarkshavn
- America/Dawson
- America/Dawson_Creek
- America/Denver
- America/Detroit
- America/Dominica
- America/Edmonton
- America/Eirunepe
- America/El_Salvador
- America/Ensenada
- America/Fort_Wayne
- America/Fortaleza
- America/Glace_Bay
- America/Godthab
- America/Goose_Bay
- America/Grand_Turk
- America/Grenada
- America/Guadeloupe
- America/Guatemala
- America/Guayaquil
- America/Guyana
- America/Halifax
- America/Havana
- America/Hermosillo
- America/Indiana/Indianapolis
- America/Indiana/Knox
- America/Indiana/Marengo
- America/Indiana/Petersburg
- America/Indiana/Tell_City
- America/Indiana/Vevay
- America/Indiana/Vincennes
- America/Indiana/Winamac
- America/Indianapolis
- America/Inuvik
- America/Iqaluit
- America/Jamaica
- America/Jujuy
- America/Juneau
- America/Kentucky/Louisville
- America/Kentucky/Monticello
- America/Knox_IN
- America/Kralendijk
- America/La_Paz
- America/Lima
- America/Los_Angeles
- America/Louisville
- America/Lower_Princes
- America/Maceio
- America/Managua
- America/Manaus
- America/Marigot
- America/Martinique
- America/Matamoros
- America/Mazatlan
- America/Mendoza
- America/Menominee
- America/Merida
- America/Metlakatla
- America/Mexico_City
- America/Miquelon
- America/Moncton
- America/Monterrey
- America/Montevideo
- America/Montreal
- America/Montserrat
- America/Nassau
- America/New_York
- America/Nipigon
- America/Nome
- America/Noronha
- America/North_Dakota/Beulah
- America/North_Dakota/Center
- America/North_Dakota/New_Salem
- America/Ojinaga
- America/Panama
- America/Pangnirtung
- America/Paramaribo
- America/Phoenix
- America/Port-au-Prince
- America/Port_of_Spain
- America/Porto_Acre
- America/Porto_Velho
- America/Puerto_Rico
- America/Rainy_River
- America/Rankin_Inlet
- America/Recife
- America/Regina
- America/Resolute
- America/Rio_Branco
- America/Rosario
- America/Santa_Isabel
- America/Santarem
- America/Santiago
- America/Santo_Domingo
- America/Sao_Paulo
- America/Scoresbysund
- America/Shiprock
- America/Sitka
- America/St_Barthelemy
- America/St_Johns
- America/St_Kitts
- America/St_Lucia
- America/St_Thomas
- America/St_Vincent
- America/Swift_Current
- America/Tegucigalpa
- America/Thule
- America/Thunder_Bay
- America/Tijuana
- America/Toronto
- America/Tortola
- America/Vancouver
- America/Virgin
- America/Whitehorse
- America/Winnipeg
- America/Yakutat
- America/Yellowknife
- Antarctica/Casey
- Antarctica/Davis
- Antarctica/DumontDUrville
- Antarctica/Macquarie
- Antarctica/Mawson
- Antarctica/McMurdo
- Antarctica/Palmer
- Antarctica/Rothera
- Antarctica/South_Pole
- Antarctica/Syowa
- Antarctica/Troll
- Antarctica/Vostok
- Arctic/Longyearbyen
- Asia/Aden
- Asia/Almaty
- Asia/Amman
- Asia/Anadyr
- Asia/Aqtau
- Asia/Aqtobe
- Asia/Ashgabat
- Asia/Ashkhabad
- Asia/Baghdad
- Asia/Bahrain
- Asia/Baku
- Asia/Bangkok
- Asia/Beirut
- Asia/Bishkek
- Asia/Brunei
- Asia/Calcutta
- Asia/Choibalsan
- Asia/Chongqing
- Asia/Chungking
- Asia/Colombo
- Asia/Dacca
- Asia/Damascus
- Asia/Dhaka
- Asia/Dili
- Asia/Dubai
- Asia/Dushanbe
- Asia/Gaza
- Asia/Harbin
- Asia/Hebron
- Asia/Ho_Chi_Minh
- Asia/Hong_Kong
- Asia/Hovd
- Asia/Irkutsk
- Asia/Istanbul
- Asia/Jakarta
- Asia/Jayapura
- Asia/Jerusalem
- Asia/Kabul
- Asia/Kamchatka
- Asia/Karachi
- Asia/Kashgar
- Asia/Kathmandu
- Asia/Katmandu
- Asia/Khandyga
- Asia/Kolkata
- Asia/Krasnoyarsk
- Asia/Kuala_Lumpur
- Asia/Kuching
- Asia/Kuwait
- Asia/Macao
- Asia/Macau
- Asia/Magadan
- Asia/Makassar
- Asia/Manila
- Asia/Muscat
- Asia/Nicosia
- Asia/Novokuznetsk
- Asia/Novosibirsk
- Asia/Omsk
- Asia/Oral
- Asia/Phnom_Penh
- Asia/Pontianak
- Asia/Pyongyang
- Asia/Qatar
- Asia/Qyzylorda
- Asia/Rangoon
- Asia/Riyadh
- Asia/Saigon
- Asia/Sakhalin
- Asia/Samarkand
- Asia/Seoul
- Asia/Shanghai
- Asia/Singapore
- Asia/Taipei
- Asia/Tashkent
- Asia/Tbilisi
- Asia/Tehran
- Asia/Tel_Aviv
- Asia/Thimbu
- Asia/Thimphu
- Asia/Tokyo
- Asia/Ujung_Pandang
- Asia/Ulaanbaatar
- Asia/Ulan_Bator
- Asia/Urumqi
- Asia/Ust-Nera
- Asia/Vientiane
- Asia/Vladivostok
- Asia/Yakutsk
- Asia/Yekaterinburg
- Asia/Yerevan
- Atlantic/Azores
- Atlantic/Bermuda
- Atlantic/Canary
- Atlantic/Cape_Verde
- Atlantic/Faeroe
- Atlantic/Faroe
- Atlantic/Jan_Mayen
- Atlantic/Madeira
- Atlantic/Reykjavik
- Atlantic/South_Georgia
- Atlantic/St_Helena
- Atlantic/Stanley
- Australia/ACT
- Australia/Adelaide
- Australia/Brisbane
- Australia/Broken_Hill
- Australia/Canberra
- Australia/Currie
- Australia/Darwin
- Australia/Eucla
- Australia/Hobart
- Australia/LHI
- Australia/Lindeman
- Australia/Lord_Howe
- Australia/Melbourne
- Australia/NSW
- Australia/North
- Australia/Perth
- Australia/Queensland
- Australia/South
- Australia/Sydney
- Australia/Tasmania
- Australia/Victoria
- Australia/West
- Australia/Yancowinna
- BET
- BST
- Brazil/Acre
- Brazil/DeNoronha
- Brazil/East
- Brazil/West
- CAT
- CET
- CNT
- CST
- CST6CDT
- CTT
- Canada/Atlantic
- Canada/Central
- Canada/East-Saskatchewan
- Canada/Eastern
- Canada/Mountain
- Canada/Newfoundland
- Canada/Pacific
- Canada/Saskatchewan
- Canada/Yukon
- Chile/Continental
- Chile/EasterIsland
- Cuba
- EAT
- ECT
- EET
- EST
- EST5EDT
- Egypt
- Eire
- Etc/GMT
- Etc/GMT+0
- Etc/GMT+1
- Etc/GMT+10
- Etc/GMT+11
- Etc/GMT+12
- Etc/GMT+2
- Etc/GMT+3
- Etc/GMT+4
- Etc/GMT+5
- Etc/GMT+6
- Etc/GMT+7
- Etc/GMT+8
- Etc/GMT+9
- Etc/GMT-0
- Etc/GMT-1
- Etc/GMT-10
- Etc/GMT-11
- Etc/GMT-12
- Etc/GMT-13
- Etc/GMT-14
- Etc/GMT-2
- Etc/GMT-3
- Etc/GMT-4
- Etc/GMT-5
- Etc/GMT-6
- Etc/GMT-7
- Etc/GMT-8
- Etc/GMT-9
- Etc/GMT0
- Etc/Greenwich
- Etc/UCT
- Etc/UTC
- Etc/Universal
- Etc/Zulu
- Europe/Amsterdam
- Europe/Andorra
- Europe/Athens
- Europe/Belfast
- Europe/Belgrade
- Europe/Berlin
- Europe/Bratislava
- Europe/Brussels
- Europe/Bucharest
- Europe/Budapest
- Europe/Busingen
- Europe/Chisinau
- Europe/Copenhagen
- Europe/Dublin
- Europe/Gibraltar
- Europe/Guernsey
- Europe/Helsinki
- Europe/Isle_of_Man
- Europe/Istanbul
- Europe/Jersey
- Europe/Kaliningrad
- Europe/Kiev
- Europe/Lisbon
- Europe/Ljubljana
- Europe/London
- Europe/Luxembourg
- Europe/Madrid
- Europe/Malta
- Europe/Mariehamn
- Europe/Minsk
- Europe/Monaco
- Europe/Moscow
- Europe/Nicosia
- Europe/Oslo
- Europe/Paris
- Europe/Podgorica
- Europe/Prague
- Europe/Riga
- Europe/Rome
- Europe/Samara
- Europe/San_Marino
- Europe/Sarajevo
- Europe/Simferopol
- Europe/Skopje
- Europe/Sofia
- Europe/Stockholm
- Europe/Tallinn
- Europe/Tirane
- Europe/Tiraspol
- Europe/Uzhgorod
- Europe/Vaduz
- Europe/Vatican
- Europe/Vienna
- Europe/Vilnius
- Europe/Volgograd
- Europe/Warsaw
- Europe/Zagreb
- Europe/Zaporozhye
- Europe/Zurich
- GB
- GB-Eire
- GMT
- GMT0
- Greenwich
- HST
- Hongkong
- IET
- IST
- Iceland
- Indian/Antananarivo
- Indian/Chagos
- Indian/Christmas
- Indian/Cocos
- Indian/Comoro
- Indian/Kerguelen
- Indian/Mahe
- Indian/Maldives
- Indian/Mauritius
- Indian/Mayotte
- Indian/Reunion
- Iran
- Israel
- JST
- Jamaica
- Japan
- Kwajalein
- Libya
- MET
- MIT
- MST
- MST7MDT
- Mexico/BajaNorte
- Mexico/BajaSur
- Mexico/General
- NET
- NST
- NZ
- NZ-CHAT
- Navajo
- PLT
- PNT
- PRC
- PRT
- PST
- PST8PDT
- Pacific/Apia
- Pacific/Auckland
- Pacific/Chatham
- Pacific/Chuuk
- Pacific/Easter
- Pacific/Efate
- Pacific/Enderbury
- Pacific/Fakaofo
- Pacific/Fiji
- Pacific/Funafuti
- Pacific/Galapagos
- Pacific/Gambier
- Pacific/Guadalcanal
- Pacific/Guam
- Pacific/Honolulu
- Pacific/Johnston
- Pacific/Kiritimati
- Pacific/Kosrae
- Pacific/Kwajalein
- Pacific/Majuro
- Pacific/Marquesas
- Pacific/Midway
- Pacific/Nauru
- Pacific/Niue
- Pacific/Norfolk
- Pacific/Noumea
- Pacific/Pago_Pago
- Pacific/Palau
- Pacific/Pitcairn
- Pacific/Pohnpei
- Pacific/Ponape
- Pacific/Port_Moresby
- Pacific/Rarotonga
- Pacific/Saipan
- Pacific/Samoa
- Pacific/Tahiti
- Pacific/Tarawa
- Pacific/Tongatapu
- Pacific/Truk
- Pacific/Wake
- Pacific/Wallis
- Pacific/Yap
- Poland
- Portugal
- ROK
- SST
- Singapore
- SystemV/AST4
- SystemV/AST4ADT
- SystemV/CST6
- SystemV/CST6CDT
- SystemV/EST5
- SystemV/EST5EDT
- SystemV/HST10
- SystemV/MST7
- SystemV/MST7MDT
- SystemV/PST8
- SystemV/PST8PDT
- SystemV/YST9
- SystemV/YST9YDT
- Turkey
- UCT
- US/Alaska
- US/Aleutian
- US/Arizona
- US/Central
- US/East-Indiana
- US/Eastern
- US/Hawaii
- US/Indiana-Starke
- US/Michigan
- US/Mountain
- US/Pacific
- US/Pacific-New
- US/Samoa
- UTC
- Universal
- VST
- W-SU
- WET
- Zulu
