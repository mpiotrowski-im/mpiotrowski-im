---
title: Moduły Javy - prawdziwe puzzle na 10000 kawałków
date: 2019-03-21
lang: pl
ref: real_jigsaw
summary: System modułów Javy jako element ekosystemu Javy jest obecnie ekstremalnie niedojrzały. Dowodzi tego m. in. moja próba zastosowania go w praktyce. Próba zakończona porażką...
---
# Moduły Javy -- prawdziwe puzzle na 10000 kawałków
Miałem do napisania drobną aplikację z interfejsem użytkownika. Do napisania aplikacji postanowiłem użyć Java 11 a do stworzenia UI zdecydowałem się użyć Java FX 11. Pomyślałem sobie: to dobra okazja aby zastosować moduły Javy w praktyce.

Rozpocząłem od zbudowania prototypu UI. Przygotowałem plik *module-info.java*. W pliku umieściłem deklaracje z jakich bibliotek JavaFX chcę skorzystać, jakie swoje pakiety otwieram. Wszystko pięknie działało i byłem dumny z siebie.

Wtedy nadszedł moment, w którym potrzebowałem użyć pewnej rozbudowanej biblioteki Open Source. To był moment, w którym zakończyła się moja przygoda z pisaniem aplikacji zgodnej z systemem modułów. Ta biblioteka, jak większość od dawna istniejących bibliotek, nie miała zadeklarowanych modułów a podział biblioteki na oddzielne JAR-y (API i implementację) powodował, że w różnych JAR-ach znajdowały się pakiety o tych samych nazwach. To niestety wyklucza użycie takiej biblioteki w trybie systemu modułów. Otrzymałem komunikat typu: `module reads package a.package from both a.impl and a.api`. Pozostało mi wyrzucić na razie pomysł zrobienia aplikacji modułowej...

Biblioteka, z której potrzebowałem skorzystać jest obecnie przepisywana na zgodną z systemem modułów, ale prace jeszcze nie zostały zakończone. Potrafię sobie jednak wyobrazić frustrację związaną z tą niewdzięczną pracą...

Podsumowując: utwiedziłem się w przekonaniu, że system modułów Javy w obecnej formie i stanie to pomyłka. Wprowadził tylko bałagan w zamian oferując niewiele (mój główny zarzut -- brak obsługi wersjonowania modułów). Potrafię sobie wyobrazić, że istnieje tysiące wartościowych bibliotek, które z tego samego powodu nie będą mogły zostać użyte z systemem modułów. Podział biblioteki na API i implementację przy jednoczesnym wykorzystaniu tych samych nazw pakietów często spotykałem na swojej drodze jako programisty...
<!--
Copyright 2019 Michał Piotrowski

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

   http://www.apache.org/licenses/LICENSE-2.0
-->
