---
layout: post
title: Caching mit dem ZF2 (Part 1/2)
archive: active
categories: development
tags: php zf2 cache
---

Das [Zend Framework 2 (ZF2)](http://framework.zend.com/zf2/) steht in den Startlöchern und es gibt einiges neues.

In diesem Artikel nehme ich mir die von Grund auf neu geschriebene Zend\Cache Komponente vor.
Als erster Einstieg sei einem erstmal die offizielle [Dokumentation](http://packages.zendframework.com/docs/latest/manual/en/index.html#zend-cache)
sowie die [API](http://packages.zendframework.com/docs/latest/apidoc/namespaces/Zend.Cache.html) ans Herz gelegt, aber das nur am Rande.

Die Komponente Zend\Cache ist in 2 unterschiedliche Teile aufgesplittet: `Zend\Cache\Storage` und `Zend\Cache\Pattern`.
In diesem ersten Artikel gehe ich näher auf `Zend\Cache\Storage` ein und mit dem nachfolgenden Artikel auf `Zend\Cache\Pattern`.


`Zend\Cache\Storage`
=================

Unter diesem Namensraum befindet sich eine Vielzahl von Interfaces, welche bestimmte Speichermethoden definieren.
Das Basis-Interface `Zend\Cache\Storage\StorageInterface` definiert die Mindestanforderungen (alle unterstützte
Speichersysteme implementieren dieses Interface).
Es definiert die folgenden Lese- und Schreiboperationen eines "Key-Values-Storage":

 - Leseoperationen
 -----------------
  - `getItem(string $key, boolean & $success = null, mixed & $casToken = null)`
  - `getItems(array $keys)`
  - `hasItem(string $key)`
  - `hasItems(array $keys)`
  - `getMetadata(string $key)`
  - `getMetadatas(array $keys)`

 - Schreiboperationen
 --------------------
  - `setItem(string $key, mixed $value)`
  - `setItems(array $keyValuePairs)`
  - `addItem(string $key, mixed $value)`
  - `addItems(array $keyValuePairs)`
  - `replaceItem(string $key, mixed $value)`
  - `replaceItems(array $keyValuePairs)`
  - `checkAndSetItem(mixed $token, string $key, mixed $value)`
  - `removeItem(string $key)`
  - `removeItems(array $keys)`
  - `touchItem(string $key)`
  - `touchItems(array $keys)`
  - `incrementItem(string $key)`
  - `incrementItems(array $keyValueParis)`
  - `decrementItem(string $key)`
  - `decrementItems(array $keyValueParis)`

All diese Methoden können von einem "Key-Value-Storage" unterstützt werden bzw. lassen sich auf einfache Weise emulieren.
So kann beispielweise die Methode `incrementItem()` emuliert werden, indem das jeweilige Item gelesen wird,
der Wert hochgezählt wird und das Item mit dem neu generierten Wert wieder in den Speicher geschrieben wird.

Wer schon des öfteren mit Cache-Systemen in PHP gearbeitet hat, dem müsste aufgefallen sein,
das nirgends ein Argument für eine TTL (time-to-live) definiert ist. Das liegt daran, das eine TTL angibt, wann
das jeweilige Item abläuft und nicht mehr gültig ist und somit bereits eine Verdrängungsstrategie darstellt.
Verdrängungsstrategien für Cache-Speicher gibt es viele und wird von der Komponente nicht vorgeschrieben.
Bei Cache-Speichern, welche diese Art der Verdrängung unterstützen wird die TTL vorher mittels einer Option
für die ganze Instanz gesetzt.

Die folgenden weiteren Interfaces definieren zusätzliche Operationen, welche von bestimmten Cache-Speichern unterstützt
werden und nur dort implementiert sind:

 - `AvailableSpaceCapableInterface`
  - `getAvailableSpace()`
 - `TotalSpaceCapableInterface`
  - `getTotalSpace()`
 - `ClearByNamespaceInterface`
  - `clearByNamespace($namespace)`
 - `ClearByPrefixInterface`
  - `clearByPrefix($prefix)`
 - `ClearExpiredInterface`
  - `clearExpired()`
 - `FlushableInterface`
  - `flush()`
 - `IterableInterface`
  - `getIterator()`
 - `OptimizableInterface`
  - `optimize()`
 - `TaggableInterface`
  - `setTags(string $key, array $tags)`
  - `getTags(string $key)`
  - `clearByTags(array $tags, boolean $disjunction = false)`

---------------------------------------

Doch halt! Was ist mit der Methode `getCapabilites()` und der Klasse `Zend\Cache\Storage\Capabilities`?
Die Capabilities (zu deutsch Fähigkeiten) geben an, was der Cache-Speicher unterstützt bzw. wie er intern arbeitet.
Im Gegensatz zu den oben definierten Interfaces werden für unterschiedliche Capabilities keine zusätzlichen oder geänderten Methoden benötigt.
Zudem können sich diese Capabilites teilweise durch setzten bestimmter Optionen ändern.

Zur Verfügung stehen folgende Capabilites:

 - supported datatypes
  - Gibt an, welche Datentypen gespeichert werden können und in welche Sie dabei evtl. konvertiert werden
 - supported metadata
  - Gibt an, welche zusatzinformationen durch die Methoden `getMetadata()` und `getMetadatas()` abrufbar sind
 - min TTL / max TTL
  - Gibt an, was die kleinste / größte TTL ist (falls verfügbar)
 - static TTL
  - Gibt an, ob die TTL mit dem Item gespeichert wird (TRUE),
    oder ob Sie beim Lesen mittels der aktuellen Zeit und der Zeit des Items berechnet wird (FALSE)
 - TTL precision
  - Gibt die genauigkeit der TTL an (falls verfügbar)
 - use request time
  - Gibt an, ob die aktuelle Zeit für Berechnungen herangezogen wird oder die Zeit der Serveranfrage
 - expired read
  - Gibt an, ob abgelaufene Einträge noch gelesen werden können, solange diese noch nicht gelöscht wurden
 - max. key length
  - Gibt die maximale Länge eines Cache-Keys an
 - namespace is prefix
  - Gibt an, ob die Implementation des Namespaces als Prefix implementiert wurde
 - namespace separator
  - Das Trennzeichen für den Namespace als Prefix mit dem eigentlichen Cache-Key

--------------------------------------------

`Zend\Cache\Storage\Plugin`
======================

Als Storage Plugin bezeichnen wir Implementierungen, welche zur Laufzeit einem Cache-Storage hinzugefügt und wieder entfernt werden können.
Dazu wird das Plugin instanziiert und dem Speicher mittels `addPlugin()` bekannt gemacht.
Intern wird dazu `Zend\EventManager` benutzt. Der speicher benutzt den `Zend\EventManager` und ruft vor und nach jeder Operation
ein "*.pre" bzw. "*.port" Event auf. Das Plugin lauscht auf diesen Events und kann dabei die Argumente und das Ergebnis abändern oder
auch einfach nur ein Lgging durchfüren.

Das Zend Framework 2 hat bereits einige Storage-Plugins Vorimplementiert. Die wohl wichtigsten sind das Serializer-Plugin und das ExceptionHandler-Plugin.

Mit Hilfe des `Zend\Cache\Storage\Plugin\Serializer`-Plugins werden alle Werte mittel eines `Zend\Serializer`'s serialisiert, so dass ansonsten nicht
unterstützte Datentypen im Cache-Speicher abgelegt werden können.

Das Plugin `Zend\Cache\Storage\Plugin\ExceptionHandler` fängt alle von dem Cache-Speicher geworfenen Exceptions ab und ruft eine definierbare
Funktion mit dieser Exception als Argument auf, um ein Logging zu implementieren.
Desweiteren kann definiert werden, ob die abgefangene Exception weitergereicht wird (re-throw),
oder ob die Operation einen einfachen Fehler als Return-Value zurück gibt (z.B. NULL bei getItem), um try-catch-Blöcke zu umgehen.

***ACHTUNG:*** Das Plugin-System ist nicht im Interface definiert und es ist daher nicht sicher, das ein Cache-Speicher dieses Unterstützt.
Mittels des Interfaces `Zend\EventManager\EventsCapableInterface` kann zwar festgestellt werden, ob der Cache-Speicher Events handhabt,
aber welche das sind ist nicht definiert.

Coding
=====
	
Nun wollen wir die Komponente aber endlich in Aktion sehen!
Dazu sollten wir sie instanziieren:

	use Zend\Cache\StorageFactory as CacheStorageFactory;
	
	$cache = CacheStorageFactory::factory(array(
		'adapter' => array(
			'name'    => 'filesystem',
			'options' => array(
				'cache_dir' => __DIR__ . '/cache',
				'ttl'       => 3600,
			),
		),
		'plugins' => array(
			'serializer' => array(
				'serializer' => 'igbinary'
			),
			'exception_handler' => array(
				'throw_exceptions'   => false,
				'exception_callback' => function (Exception $e) {
					$message = date('r') . ' ' . $e . PHP_EOL;
					file_put_contents(__DIR__ . '/log/cache_error.log', $message, FILE_APPEND);
				},
			),
		),
	));


-----------------------------------------------

Mit dem noch folgenden Artikel werde ich einen Einblick in den Namensraum von `Zend\Cache\Pattern` geben.

Bis hierher erstmal happy coding :)
