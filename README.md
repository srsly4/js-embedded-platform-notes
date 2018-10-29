# Js embedded platform dev notes
## Eventloop
W prezentowanych przez Duktape przykładach eventloop rozróżniał typy wywołań asynchronicznych - osobne obiekty na wywołania timerów, na polling deskryptorów plików.

**Pomysł** - zastąpienie wielu obiektów jednym, zunifikowanym obiektem zawierającym odniesienia do funkcji.

Pierwotne podejście: inicjalizacja globalnego obiektu przechowującego funkcje callback aby uniknąć ich usunięcia przez GC. Callbacki obecne jako struktura w pamięci FreeRTOS oraz jako funkcja w obiektcie na stosie Duktape identyfikowana przez sekwencyjny `uint64_t` (*potencjalny problem z rzutowaniem na IEEE Double?*). Struktura callbacka:

 - identyfikator (uint64)
 - `user_data` - wskaźnik do wykorzystania przez implementującego
 - `arg_handler` - wskaźnik do funkcji wykorzystywanej do przekazania argumentów callbacka (przed wykonaniem)
 - `completion_handler` - wskaźnik do funkcji wykonywanej po wykonaniu callbacka

Skorzystanie z mechanizmu kolejek oferowanego przez FreeRTOS. Wątek eventloopa oczekuje na nadchodzace callbacki:

 1. Poczekaj na callback i pobierz go
 2. Znajdź na stercie Duktape odpowiednią funkcję
 3. Wrzuć ją na stos wywołań
 4. Wykonaj `argument_handler` pozwalający implementującemu funkcję dodanie własnych argumentów do callbacków (inversion of control)
 5. Wykonaj wrzuconą funkcję poprzez interpreter Duktape (pcall, może użyci safe_pcall i chwytać błędy?)
 6. Wykonaj `completion_handler` - pozwalający inicjującemu callback na uwolnienie niepotrzebnej pamięci (w setTimeout - deinicjalizację timera)

### setTimeout/setInterval
Niezbędne zestaw funkcji w asynchronicznym JS:
* setTimeout
* setInterval
* clearTimeout
* clearInterval

**Pomysł** - realizacja poprzez wykorzystanie timerów systemu FreeRTOS. Posiadają wbudowaną flagę określającą timer jako auto-shot/one-shot. Funkcja `setTimeout`/`setInterval` tworzy timer, który po upływie czasu dodaje callback do kolejki wywołań.

**Problem** - w jaki sposób zarządzać pamięcia wykorzystanych timerów? Obecnie: wykorzystanie `completion_handler` i wykonywanie callback_destroy

**Problem** - po załadowaniu nowego kodu timery poprzedniego dalej są aktywne
**Rozwiązanie** - stworzenie double-linked listy dla timerów, opakowanie callbacków w strukturę `timer_item_t` (tylko na platformie STM). Lista jest na bieżąco uaktualniania o nowe timery i te, które upłynęły bądź zostały zatrzymane. Przed restartem interpretera z nowym kodem wszystkie timery są zatrzymywane i usuwane przez funkcję `..._cleanup`.

**Problem** - "dynamiczne tworzenie timerów w timerach" powoduje po dłuższym czasie (~2-3 min) niesprecyzowany błąd pamięci . Przykładowy kod dla interpretera:
```javascript
(function(){
    ledOff();
    var sw = false;
    setInterval(function(){
        var inter = setInterval(function(){
            if (!sw) { ledOn(); } else { ledOff(); }
            sw = !sw;
        }, 100);
        setTimeout(function(){
            clearInterval(inter);
            ledOff();
            sw=false;
        }, 1500)
    }, 2500);
})()
``` 
Co 2,5s tworzony jest nowy timer.
**Hipoteza** - błędne zarządzanie pamięcią timerów bądź nieprawidłowa obsługa dynamicznych timerów systemów FreeRTOS

Obserwacja zużycia pamięci:
 * początkowo wolne ~8 KiB
 * w każdej sekundzie odświeżania podstrony /heap ilość wolnej pamięcia zmniejsza się
 * przy około 800 B wolnej pamięci następuję skok ponownie do 8 KiB
 * błąd nie występuję przy jakiejś konkretnej ilości wolnej pamięci

**Wniosek** - ilość wolnej pamięci nie ma wpływu na błąd

**Hipoteza** - błędna konfiguracja timerów FreeRTOS
Wykonane czynności:
 * Pierwotna wartość `configTIMER_TASK_STACK_DEPTH` była równa 128 słowom.
 * Callback timera wywołuje tylko wrzucenie obiektu `callback_t` do kolejki wywołań. Sugestia obniżenia tej wartości celem oszczędzenia pamięci.
 * Obniżenie wartości do 32 słów.
 * **Zatrzymanie wykonania następuje już po kilku sekundach**
 * Zwiększenie wartości do 256 słów.
 * **Brak zatrzymania wykonania przez 30 minut** - założenie rozwiązania problemów

**Pytanie** - *dlaczego callback timera potrzebuje tak dużego stosu i dlaczego jego wymagany rozmiar zmienia się z czasem?*

### Stabilność
**Obserwacja** - stabilność zarządzania pamięcia dla interpretera asynchronicznego z obsługą timerów.
 * Pomysł - wykonywanie zapytania HTTP /heap co 1 sekundę
 * Niestety, program powoduje niemal od razu zatrzymanie programu na STM (*problemy z pamięcią?*)
 * Pomysł - wykorzystanie pakietów *Hello there* z broadcast UDP
 * **Obserwacja** - wykonywanie broadcastów co 100ms zatrzymało urządzenie po kilkudziesięciu sekundach
 * Zmiana interwału broadcastu na 1s 
* Napisanie prostego UDP broadcast listenera w node.js

Kod programu do obserwacji:
```javascript
const dgram = require('dgram');

const server = dgram.createSocket("udp4");
server.bind(9998);

server.on('message', (message) => {
    const entry = `${Date.now().toString()};${message}`;
    console.log(entry);
});
``` 
Obserwacja w interwałach 1s pokazała, że w przypadku "zamrożenia" po kilkunastu sekundach, wolna pamięć spada niemal do 0. [wykres 1, memdump2.csv]

Co ciekawe, w niektórych przypadkach program nie ulega zatrzymaniu. Po wartościach bliskich zeru, ilość wolnej pamięci wzrasta skokowo niemal do 13 KiB - czyli wartości początkowej.

**Hipoteza** - za zwolnienie zajętego RAMu w w/w obserwacji odpowiadał Garbage Collector z Duktape, który w pewnych przypadkach nie zadziała.

Dodanie wywołania `duk_gc(0)` w `_callback_destroy` poskutkowało wynikiem [wykres2, memdump3.csv] - wolna pamięć nigdy nie spada poniżej 12 KiB.

**Wniosek** - GC Duktape działa tylko wtedy, kiedy nie powiedzie się `malloc()` wykonany z poziomu Duktape. Nie zadziała natomiast, jeżeli Duktape wykorzystał sporą ilość pamięci i inny zasób chce zaalokować pamięc używając tego samego `malloc()`.

**Pomysł** -  odseparowanie puli pamięci dla Duktape i dla innych komponentów
**Pomysł** - stworzenie osobnego taska, który co pewien czas będzie wykonywał funkcję `duk_gc()`

Póki co, GC jest wywoływany przy zwalnianiu pamięci dla callbacków. Późniejsze integracje z modułami wykażą konieczność (lub jej brak) zastosowania innych mechanizmów.

Mimo zmian w działaniu GC, dalej program ulega zatrzymaniu niedeterministycznie.

Któryś z błędów spowodował wywołanie error handlera dla zdarzenia stack overflow dla tasków FreeRTOS (sygnalizowanego przez trzy mignięcia czerowną diodą LED). Przy użyciu gdb można było podejrzeć, który task spowodował przepełnienie stosu. Okazał się nim być *idle task*. Po krótkim researchu remedium na błąd okazało się zwiększenie minimalnego rozmiaru stosu dla taska. Wartość ta była wykorzystywana także właśnie jako rozmiar stosu dla *idle task*.
Obecna wartość po podniesieniu (było 32): `configMINIMAL_STACK_SIZE ((uint16_t)128)`
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTA5OTUyODQ1OF19
-->