.. highlight:: php
   :linenothreshold: 2

.. index::
   single: wydajność

Wydajność
=========

Symfony jest szybkie zaraz po uruchomieniu. Oczywiście jeśli potrzebuje się
jeszcze lepszej wydajności, jest kilka sposobów na przyśpieszenie Symfony. W tym
rozdziale prezentujemy kilka sposobów do przyśpieszenie aplikacji opartych na Symfony.
Mają one bardzo duże możliwości i są szeroko stosowane.

.. index::
   single: wydajność; akceleratory kodu bajtowego

Akceleratory skompilowanego kodu bajtowego (np. APC)
----------------------------------------------------

Jednym z najlepszych (i najłatwiejszych) sposobów zwiększenia wydajności jest
użycie **akceleratorów skompilowanego kodu bajtowego**, czyli pamięci podręcznej
przechowującej kod bajtowy. Ideą buforowania `kodu bajtowego`_  jest wyeliminowanie
potrzeby ciągłego kompilowywania tego samego kodu. Dostępnych jest kilka
`akceleratorów kodu bajtowego`_ dla PHP o otwartym kodzie.
W PHP 5.5 wbudowany został `OPcache`_. W starszych wersjach, powszechnie używany
był akcelerator kodu bajtowego `APC`_

Korzystanie z pamięci podręcznej tego typu naprawdę nie ma wad i Symfony zostało
zaprojektowane tak aby radzić sobie bardzo dobrze w tych warunkach.

Dalsze Optymalizacje
~~~~~~~~~~~~~~~~~~~~

Akceleratory kodu bajtowego zwykle monitorują zmiany w pliku źródłowym. Zapewnia to,
że jeśli plik żródłowy zostanie zmieniony, kod żródłowy zostanie automatycznie
przekompilowany do kodu bajtowego. Jest to wygodne, ale w oczywisty sposób dodaje
znaczny narzut czasowy.

Z tego powodu , niektóre akceleratory kodu bajtowego oferuję opcję do wyłączenia
tego sprawdzania. Oczywiście, gdy wyłączy się taką kontrolę, to do obowiązków
administratora systemu będzie należeć czyszczenie pamięci podręcznej, po każdej
zmianie plików źródłowych. W przeciwnym razie aktualizacja nie będzie widoczna.

Dla przykładu, aby wyłączyć w APC sprawdzanie, trzeba dodać w pliku ``php.ini`` 
linię ``apc.stat=0`` 
.

.. index::
   single: wydajność; autoloader

Mapa klas z użyciem programu Composer
-------------------------------------

Domyślnie Symfony SE używa autoloadera programu Composer w pliku `autoload.php`_.
Ten autoloader jest łatwy w użyciu, jako że automatycznie odnajduje nowe klasy,
które zostały umieszczone w zarejestrowanych katalogach.

Niestety, wiąże się to z kosztami, ponieważ auloader iteruje po wszystkich 
skonfigurowanych przestrzeniach nazw w celu znalezienia konkretnego pliku, wywołując
za każdym razem funkcję ``file_exists``.

Prostszym rozwiązaniem jest powiadomienie programu Composer aby zbudował "mapę klas"
(tj. dużą tablicę lokalizacji wszystkich klas). Można to zrealizować z wiersza
poleceń na etapie procesu wdrożenia:

.. code-block:: bash

    $ php composer.phar dump-autoload --optimize

Wewnętrznie buduje to wielką tablicę mapy klas w ``vendor/composer/autoload_namespaces.php``.


Buforowanie autoloadera w APC
-----------------------------

Innym rozwiązaniem jest buforowanie lokalizacji każdej klasy, po tym jak zostanie
po raz pierwszy umieszczona w swojej lokalizacji. Symfony dostarczane jest z klasą,
:class:`Symfony\\Component\\ClassLoader\\ApcClassLoader`, która to właśnie realizuje.
Aby jej użyć, wystarczy dostosować plik kontrolera wejścia. Jeśli używa się standardowej
dystrybucji, kod ten powinien już być dostępny w pliku w postaci komentarza::

    // app.php

    // ...
    $loader = require __DIR__.'/../app/autoload.php';
    include_once __DIR__.'/../var/bootstrap.php.cache';
    // Enable APC for autoloading to improve performance.
    // You should change the ApcClassLoader first argument to a unique prefix
    // in order to prevent cache key conflicts with other applications
    // also using APC.
    /*
    $apcLoader = new Symfony\Component\ClassLoader\ApcClassLoader(sha1(__FILE__), $loader);
    $loader->unregister();
    $apcLoader->register(true);
    */

    // ...
    

Więcej szczegółów w :doc:`/components/class_loader/cache_class_loader`.

.. note::

    Podczas używania autoloadera APC, jeśli doda się nową klasę, to zostanie ona
    odnaleziona automatycznie i wszystko będzie działać tak jak poprzednio (tj.
    nie trzeba będzie czyścić pamięci podręcznej). Jeśli jednak zmieni się lokalizację
    określonej przestrzeni nazw lub doda przedrostek, to trzeba będzie przepłukać
    pamięć podręczną APC. W przeciwnym razie autoloader będzie nadal wyszukiwał
    starą lokalizację dla wszystkich klas w przestrzeni nazw.

.. index::
   single: wydajność; pliki rozruchowe

Pliki rozruchowe
----------------

Aby zapewnić optymalną elastyczność i możliwość ponownego użycia kodu, Symfony
posiada sporą różnorodność klas oraz komponentów zewnętrznych. Ale ładowanie
tych wszystkich klas z osobnych plików przy każdym wywołaniu (request) może
dawać narzut czasowy. Aby zminimalizować ten narzut, Symfony Standard Edition 
udostępnia skrypt do wygenerowania pliku rozruchowego `bootstrap`_, który zawiera
definicję wielu klas w jednym miejscu.
Poprzez ładowanie tego pliku (który posiada kopię wielu klas z jądra), Symfony nie 
musi więcej ładować źródła plików zawierających te klasy. To trochę zredukuje 
operacje dyskowe IO.

Jeśli używa się Symfony Standard Edition, w takim przypadku zapewne używa się już
pliku rozruchowego. Aby to sprawdzić czy w kontrolerze wejścia
(zwykle ``app.php``) istnieje następująca linia::

    include_once __DIR__.'/../var/bootstrap.php.cache';

Trzeba mieć na uwadze, że używanie pliku rozruchowego posiada dwie wady:

* plik musi zostać wygenerowany ponownie gdy zmieni się jakiś plik źródłowy
  (np. kiedy robi się aktualizację kodu Symfony lub też bibliotek dostawców);

* podczas debugowania, trzeba ustawić punkty przerwania (break points) wewnątrz
  pliku rozruchowego.

Jeśli używasz Symfony Standard Edition, plik rozruchowy jest automatycznie 
przebudowywany po aktualizacji bibliotek dostawców po użyciu polecenia 
``php bin/vendors install``.

Pliki rozuchowe a akceleratory kodu bajtowego
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Nawet przy użyciu akceleratora kodu bajtowego wydajność systemu zostanie poprawiona
poprzez zastosowanie pliku rozruchoweho, ponieważ będzie mniej plików do monitorowania
zmian. Oczywiście jeśli ta funkcjonalność jest wyłączona w akceleratorze kodu bajtowego
(np. ``apc.stat=0`` w APC), to nie ma powodów aby używać pliku rozuchowego.

.. _`APC`: http://php.net/manual/en/book.apc.php
.. _`autoload.php`: https://github.com/symfony/symfony-standard/blob/master/app/autoload.php
.. _`bootstrap`: https://github.com/sensio/SensioDistributionBundle/blob/master/Resources/bin/build_bootstrap.php
.. _`kodu bajtowego`: http://pl.wikipedia.org/wiki/Kod_bajtowy
.. _`akceleratorów kodu bajtowego`: https://en.wikipedia.org/wiki/List_of_PHP_accelerators
.. _`OPcache`: http://php.net/manual/en/book.opcache.php