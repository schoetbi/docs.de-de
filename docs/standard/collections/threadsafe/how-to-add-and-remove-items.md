---
title: "Gewusst wie: Hinzufügen und Entfernen von Elementen aus einem ConcurrentDictionary"
description: "Gewusst wie: Hinzufügen und Entfernen von Elementen aus einem ConcurrentDictionary"
keywords: .NET, .NET Core
author: mairaw
ms.author: mairaw
ms.date: 06/20/2016
ms.topic: article
ms.prod: .net
ms.technology: dotnet-standard
ms.devlang: dotnet
ms.assetid: b7c04a5f-a8e6-42ae-8c84-0e1ae18896eb
translationtype: Human Translation
ms.sourcegitcommit: c15f2da15c6448cf1c36dea2d5fd53e734bb6608
ms.openlocfilehash: 3e99bcb1cfbcf0a4f6f372e481d41ec31d635749

---

# <a name="how-to-add-and-remove-items-from-a-concurrentdictionary"></a>Gewusst wie: Hinzufügen und Entfernen von Elementen aus einem ConcurrentDictionary

Dieses Beispiel zeigt das Hinzufügen, Abrufen, Aktualisieren und Entfernen von Elementen einer [System.Collections.Concurrent.ConcurrentDictionary&lt;TKey, TValue&gt;](https://docs.microsoft.com/dotnet/core/api/System.Collections.Concurrent.ConcurrentDictionary-2)-Auflistung. Diese Auflistungsklasse ist eine threadsichere Implementierung. Es empfiehlt sich, diese Klasse immer dann zu verwenden, wenn möglicherweise mehrere Threads gleichzeitig versuchen, auf Elemente zuzugreifen.

`ConcurrentDictionary<TKey, TValue>` bietet verschiedene praktische Methoden, mit denen der Code nicht mehr zuerst prüfen muss, ob ein Schlüssel vorhanden ist, bevor versucht wird, Daten hinzuzufügen oder zu entfernen. Die folgende Tabelle führt diese praktischen Methoden auf und beschreibt ihre Verwendung.

Methode | Wenn folgende Bedingungen vorliegen
------ | -----------
`AddOrUpdate` | Sie möchten einen neuen Wert für einen angegebenen Schlüssel hinzufügen und, wenn der Schlüssel bereits vorhanden ist, den Wert ersetzen.
`GetOrAdd` | Sie möchten den vorhandenen Wert für einen angegebenen Schlüssel abrufen und, wenn der Schlüssel nicht vorhanden ist, ein Schlüssel-Wert-Paar angeben.
`TryAdd`, `TryGetValue`, `TryUpdate`, `TryRemove` | Sie möchten ein Schlüssel-Wert-Paar hinzufügen, abrufen, aktualisieren oder entfernen und, wenn der Schlüssel bereits vorhanden ist oder der Versuch aus einem anderen Grund fehlschlägt, eine alternative Aktion durchführen.

## <a name="example"></a>Beispiel

```csharp
namespace DictionaryHowTo
{
    using System;
    using System.Collections.Concurrent;
    using System.Collections.Generic;
    using System.Linq;
    using System.Text;
    using System.Threading;
    using System.Threading.Tasks;

    // The type of the Value to store in the dictionary:
    class CityInfo : IEqualityComparer<CityInfo>
    {
        public string Name { get; set; }
        public DateTime lastQueryDate { get; set; }
        public decimal Longitude { get; set; }
        public decimal Latitude { get; set; }
        public int[] RecentHighTemperatures { get; set; }

        public CityInfo(string name, decimal longitude, decimal latitude, int[] temps)
        {
            Name = name;
            lastQueryDate = DateTime.Now;
            Longitude = longitude;
            Latitude = latitude;
            RecentHighTemperatures = temps;
        }

        public CityInfo()
        {
        }

        public CityInfo(string key)
        {
            Name = key;
            // MaxValue means "not initialized"
            Longitude = Decimal.MaxValue;
            Latitude = Decimal.MaxValue;
            lastQueryDate = DateTime.Now;
            RecentHighTemperatures = new int[] { 0 };

        }
        public bool Equals(CityInfo x, CityInfo y)
        {
            return x.Name == y.Name && x.Longitude == y.Longitude && x.Latitude == y.Latitude;
        }

        public int GetHashCode(CityInfo obj)
        {
            CityInfo ci = (CityInfo)obj;
            return ci.Name.GetHashCode();
        }
    }

    class Program
    {
        // Create a new concurrent dictionary.
        static ConcurrentDictionary<string, CityInfo> cities = new ConcurrentDictionary<string, CityInfo>();

        static void Main(string[] args)
        {
            CityInfo[] data = 
            {
                new CityInfo(){ Name = "Boston", Latitude = 42.358769M, Longitude = -71.057806M, RecentHighTemperatures = new int[] {56, 51, 52, 58, 65, 56,53}},
                new CityInfo(){ Name = "Miami", Latitude = 25.780833M, Longitude = -80.195556M, RecentHighTemperatures = new int[] {86,87,88,87,85,85,86}},
                new CityInfo(){ Name = "Los Angeles", Latitude = 34.05M, Longitude = -118.25M, RecentHighTemperatures =   new int[] {67,68,69,73,79,78,78}},
                new CityInfo(){ Name = "Seattle", Latitude = 47.609722M, Longitude =  -122.333056M, RecentHighTemperatures =   new int[] {49,50,53,47,52,52,51}},
                new CityInfo(){ Name = "Toronto", Latitude = 43.716589M, Longitude = -79.340686M, RecentHighTemperatures =   new int[] {53,57, 51,52,56,55,50}},
                new CityInfo(){ Name = "Mexico City", Latitude = 19.432736M, Longitude = -99.133253M, RecentHighTemperatures =   new int[] {72,68,73,77,76,74,73}},
                new CityInfo(){ Name = "Rio de Janiero", Latitude = -22.908333M, Longitude = -43.196389M, RecentHighTemperatures =   new int[] {72,68,73,82,84,78,84}},
                new CityInfo(){ Name = "Quito", Latitude = -0.25M, Longitude = -78.583333M, RecentHighTemperatures =   new int[] {71,69,70,66,65,64,61}}
            };

            // Add some key/value pairs from multiple threads.
            Task[] tasks = new Task[2];

            tasks[0] = Task.Run(() =>
            {
                for (int i = 0; i < 2; i++)
                {
                    if (cities.TryAdd(data[i].Name, data[i]))
                        Console.WriteLine("Added {0} on thread {1}", data[i],
                            Thread.CurrentThread.ManagedThreadId);
                    else 
                        Console.WriteLine("Could not add {0}", data[i]);
                }
            });

            tasks[1] = Task.Run(() =>
            {
                for (int i = 2; i < data.Length; i++)
                {
                    if (cities.TryAdd(data[i].Name, data[i]))
                        Console.WriteLine("Added {0} on thread {1}", data[i],
                            Thread.CurrentThread.ManagedThreadId);
                    else
                        Console.WriteLine("Could not add {0}", data[i]);
                }
            });

            // Output results so far.
            Task.WaitAll(tasks);

            // Enumerate collection from the app main thread.
            // Note that ConcurrentDictionary is the one concurrent collection
            // that does not support thread-safe enumeration.
            foreach (var city in cities)
            {
                Console.WriteLine("{0} has been added.", city.Key);
            }

            AddOrUpdateWithoutRetrieving();
            RetrieveValueOrAdd();
            RetrieveAndUpdateOrAdd();  

            Console.WriteLine("Press any key.");
            Console.ReadKey();
        }

        // This method shows how to add key-value pairs to the dictionary
        // in scenarios where the key might already exist.
        private static void AddOrUpdateWithoutRetrieving()
        {
            // Sometime later. We receive new data from some source.
            CityInfo ci = new CityInfo() { Name = "Toronto",
                                            Latitude = 43.716589M,
                                            Longitude = -79.340686M,
                                            RecentHighTemperatures = new int[] { 54, 59, 67, 82, 87, 55, -14 } };

            // Try to add data. If it doesn't exist, the object ci is added. If it does
            // already exist, update existingVal according to the custom logic in the 
            // delegate.
            cities.AddOrUpdate(ci.Name, ci,
                (key, existingVal) =>
                {
                    // If this delegate is invoked, then the key already exists.
                    // Here we make sure the city really is the same city we already have.
                    // (Support for multiple cities of the same name is left as an exercise for the reader.)
                    if (ci != existingVal)
                        throw new ArgumentException("Duplicate city names are not allowed: {0}.", ci.Name);

                    // The only updatable fields are the temerature array and lastQueryDate.
                    existingVal.lastQueryDate = DateTime.Now;
                    existingVal.RecentHighTemperatures = ci.RecentHighTemperatures;
                    return existingVal;
                });

            // Verify that the dictionary contains the new or updated data.
            Console.Write("Most recent high temperatures for {0} are: ", cities[ci.Name].Name);
            int[] temps = cities[ci.Name].RecentHighTemperatures;
            foreach (var temp in temps) Console.Write("{0}, ", temp);
            Console.WriteLine();
        }

        // This method shows how to use data and ensure that it has been
        // added to the dictionary.
        private static void RetrieveValueOrAdd()
        {
            string searchKey = "Caracas";
            CityInfo retrievedValue = null;

            try
            {
                retrievedValue = cities.GetOrAdd(searchKey, GetDataForCity(searchKey));
            }
            catch (ArgumentException e)
            {
                Console.WriteLine(e.Message);
            }

            // Use the data.
            if (retrievedValue != null)
            {
                Console.Write("Most recent high temperatures for {0} are: ", retrievedValue.Name);
                int[] temps = cities[retrievedValue.Name].RecentHighTemperatures;
                foreach (var temp in temps) Console.Write("{0}, ", temp);
            }
            Console.WriteLine();
        }




        // This method shows how to retrieve a value from the dictionary,
        // when you expect that the key/value pair already exists,
        // and then possibly update the dictionary with a new value for the key.
        private static void RetrieveAndUpdateOrAdd()
        {
            CityInfo retrievedValue;
            string searchKey = "Buenos Aires";

            if (cities.TryGetValue(searchKey, out retrievedValue))
            {
                // use the data
                Console.Write("Most recent high temperatures for {0} are: ", retrievedValue.Name);
                int[] temps = retrievedValue.RecentHighTemperatures;
                foreach (var temp in temps) Console.Write("{0}, ", temp);

                // Make a copy of the data. Our object will update its lastQueryDate automatically.
                CityInfo newValue = new CityInfo(retrievedValue.Name,
                                                retrievedValue.Longitude,
                                                retrievedValue.Latitude,
                                                retrievedValue.RecentHighTemperatures);

                // Replace the old value with the new value.
                if (!cities.TryUpdate(searchKey, retrievedValue, newValue))
                {
                    //The data was not updated. Log error, throw exception, etc.
                    Console.WriteLine("Could not update {0}", retrievedValue.Name);
                }
            }
            else
            {
                // Add the new key and value. Here we call a method to retrieve
                // the data. Another option is to add a default value here and 
                // update with real data later on some other thread.
                CityInfo newValue = GetDataForCity(searchKey);
                if( cities.TryAdd(searchKey, newValue))
                {
                    // use the data
                    Console.Write("Most recent high temperatures for {0} are: ", newValue.Name);
                    int[] temps = newValue.RecentHighTemperatures;
                    foreach (var temp in temps) Console.Write("{0}, ", temp);
                }
                else
                    Console.WriteLine("Unable to add data for {0}", searchKey);
            }
        }

        //Assume this method knows how to find long/lat/temp info for any specified city.
        static CityInfo GetDataForCity(string name)
        {
            // Real implementation left as exercise for the reader.
            if (String.CompareOrdinal(name, "Caracas") == 0)
                return new CityInfo() { Name = "Caracas", 
                                        Longitude = 10.5M, 
                                        Latitude = -66.916667M,
                                        RecentHighTemperatures = new int[] { 91, 89, 91, 91, 87, 90, 91 } };
            else if (String.CompareOrdinal(name, "Buenos Aires") == 0)
                return new CityInfo() { Name = "Buenos Aires", 
                                        Longitude = -34.61M, 
                                        Latitude = -58.369997M, 
                                        RecentHighTemperatures = new int[] { 80, 86, 89, 91, 84, 86, 88 } };
            else
                throw new ArgumentException("Cannot find any data for {0}", name);
        }
    }
}

```
[ConcurrentDictionary&lt;TKey, TValue&gt;](https://docs.microsoft.com/dotnet/core/api/System.Collections.Concurrent.ConcurrentDictionary-2) ist für Multithreadszenarien konzipiert. Sie müssen keine Sperren in Ihrem Code verwenden, um Elemente zur Auflistung hinzuzufügen oder daraus zu entfernen. Es ist jedoch immer möglich, dass ein Thread einen Wert abruft und ein anderer Thread durch Zuweisen eines neuen Werts zum gleichen Schlüssel die Auflistung unmittelbar danach aktualisiert.

Darüber hinaus sind zwar alle Methoden von `ConcurrentDictionary<TKey, TValue>` threadsicher, aber nicht alle Methoden sind atomisch – insbesondere `GetOrAdd` und `AddOrUpdate`. Der an diese Methoden übergebene Benutzerdelegat wird außerhalb der internen Sperre des Wörterbuchs aufgerufen. (Dies erfolgt, um zu verhindern, dass unbekannter Code alle Threads blockiert.) Daher ist es möglich, dass die folgende Ereignissequenz eintritt:

1. threadA ruft `GetOrAdd` auf, findet kein Element und erstellt ein neues hinzuzufügendes Element durch Aufrufen des valueFactory-Delegaten.

2. threadB ruft `GetOrAdd` gleichzeitig auf, der zugehörige valueFactory-Delegat wird aufgerufen und erreicht die interne Sperre vor threadA. Daher wird das neue Schlüssel-Wert-Paar dieses Threads zum Wörterbuch hinzugefügt.

3. Die Verarbeitung des Benutzerdelegaten von threadA wird abgeschlossen, der Thread erreicht die Sperre und stellt jetzt fest, dass das Element bereits vorhanden ist.

4. threadA führt einen „Get“-Vorgang aus und gibt die Daten zurück, die vorher von threadB hinzugefügt wurden.

Daher ist nicht garantiert, dass die von `GetOrAdd` zurückgegebenen Daten die gleichen Daten sind, die von der valueFactory des Threads erstellt wurden. Eine ähnliche Abfolge von Ereignissen kann eintreten, wenn `AddOrUpdate` aufgerufen wird. 

## <a name="see-also"></a>Siehe auch

[System.Collections.Concurrent](https://docs.microsoft.com/dotnet/core/api/System.Collections.Concurrent)

[Threadsichere Sammlungen](index.md)




<!--HONumber=Nov16_HO3-->


