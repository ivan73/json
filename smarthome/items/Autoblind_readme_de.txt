
[beispiel]
    [[steckdosen]]
        type = bool

    [[licht]]
        type = bool

    [[wetterstation]]
        [[[helligkeit]]]
            name = Helligkeit
            type = num

            [[[[gt43k]]]]
                type = bool
                name = Flag: Helligkeit größer als 43 kLux
                eval = sh.beispiel.wetterstation.helligkeit() > 43000
                eval_trigger beispiel.wetterstation.helligkeit

            [[[[gt35k]]]]
                type = bool
                name = Flag: Helligkeit größer als 35 kLux
                eval = sh.beispiel.wetterstation.helligkeit() > 35000
                eval_trigger beispiel.wetterstation.helligkeit

            [[[[gt25k]]]]
                type = bool
                name = Flag: Helligkeit größer als 25 kLux
                eval = sh.beispiel.wetterstation.helligkeit() > 25000
                eval_trigger beispiel.wetterstation.helligkeit

            [[[[gt20k]]]]
                type = bool
                name = Flag: Helligkeit größer als 20 kLux
                eval = sh.beispiel.wetterstation.helligkeit() > 20000
                eval_trigger beispiel.wetterstation.helligkeit

        [[[temperatur]]]
        name = Temperatur
        type = num
Trigger

Da wir mehrere Raffstores automatisieren wollen und alle Raffstores gleichzeitig fahren sollen, brauchen wir einen externen Trigger, auf den dann alle Automatiken hören:

[beispiel]
    [[trigger]]
        [[[raffstore]]]
            type = bool
            name = Gemeinsamer Trigger für alle Raffstores
            enforce_updates = yes
            cycle = 300 = 1
In diesem Fall wird die Zustandsermittlung alle 300 Sekunden (5 Minuten) ausgelöst.

Default-Konfiguration

Nun kommt die Default-Konfiguration. Sie ist unabhängig von konkreten zu automatisierenden Objekten. Sie beinhaltet jedoch umfangreiche Einstellungen, so dass die zu automatisierenden Objekte, die die Einstellungen aus der Default-Konfiguration verwenden, oft sehr simpel aufgebaut werden können.

[beispiel]
    [[default]]    
        [[[raffstore]]]
            # Item für Helligkeit außen
            as_item_brightness = beispiel.wetterstation.helligkeit
            # Item für Temperatur außen
            as_item_temperature = beispiel.wetterstation.temperatur
            # Item das anzeigt, ob die Helligkeit außen mehr als 25kLux beträgt
            as_item_brightnessGt25k = beispiel.wetterstation.helligkeit.gt25k
            # Item das anzeigt, ob die Helligkeit außen mehr als 43kLux beträgt
            as_item_brightnessGt43k = beispiel.wetterstation.helligkeit.gt43k
            # Item für Behanghöhe
            as_item_hoehe = ...hoehe
            # Keine Änderung der Behanghöhe wenn Abweichung kleiner 10
            as_mindelta_hoehe = 10
            # Item für Lamellenwinkel          
            as_item_lamelle = ...lamelle
            # Keine Änderung des Lamellenwinkels wenn Abweichung kleiner 5
            as_mindelta_lamelle = 5
            # "Manuell" Item
            as_item_manuell = ..manuell
            # "Lock" Item
            as_item_lock = ..lock
            # "Suspend" Item
            as_item_suspend = ..suspend

            # Zustand "Sperre über Sperr-Item"
            [[[[Lock]]]]
                type = foo
                name = Automatik manuell gesperrt
                # Aktionen:
                # - "Suspend"-Item ggf. zurücksetzen               
                as_set_suspend = False
                # sonst nichts tun
                [[[[[enter]]]]]
                    # Einstieg in "Lock": Wenn
                    # - das "Lock"-Item gesetzt ist
                    as_value_lock = True

            # Zustand "Zeitweises Deaktivieren bei manuellen Aktionen"
            [[[[Suspend]]]]
                type = foo
                name = Ausgesetzt
                # Namensermittlung über eval-Funktion
                as_name = eval:autoblind_eval.insert_suspend_time("..suspend", "Automatik ausgesetzt bis %X")
                # Aktionen:
                # - "Suspend"-Item setzen
                as_set_suspend = True
                # sonst nichts tun

                # Einstieg in "Suspend": Wenn 
                [[[[[enter_manuell]]]]]                        
                    # - die Zustandsermittlung über das "Manuell"-Item ausgelöst wurde
                    as_value_trigger_source = eval: autoblind_eval.get_relative_itemid("..manuell")

                # Verbleib in "Suspend": Wenn
                [[[[[enter_stay]]]]]
                    # - wir bereits in "Suspend" sind
                    as_value_laststate = var:current.state_id
                    # - die letzte Änderung des "Manuell"-Items höchstens 3600 Sekunden her ist
                    as_agemax_manuell = 3600
                    # - das "Suspend"-Item nicht von irgendwo anders auf "False" gesetzt wurde
                    as_value_suspend = True

            # Zustand "Nachführen der Lamellen zum Sonnenstand bei großer Helligkeit", Gebäudeseite 1
            [[[[Nachfuehren_Seite_Eins]]]]
                type = foo
                name = Tag (nachführen)
                # Aktionen:
                # - Behang ganz herunterfahren
                as_set_hoehe = value:100
                # - Lamellen zur Sonne ausrichten
                as_set_lamelle = eval:autoblind_eval.sun_tracking()
                # - "Suspend"-Item ggf. zurücksetzen               
                as_set_suspend = False

                # Einstieg in "Nachführen": Wenn 
                [[[[[enter]]]]]
                    # - das Flag "Helligkeit > 43kLux" seit mindestens 60 Sekunden gesetzt ist
                    as_value_brightnessGt43k = true
                    as_minage_brightnessGt43k = 60
                    # - die Sonnenhöhe mindestens 18° ist
                    as_min_sun_altitude = 18
                    # - die Sonne aus Richtung 130° bis 270° kommt
                    as_min_sun_azimut = 130
                    as_max_sun_azimut = 270
                    # - es draußen mindestens 22° hat
                    as_min_temperature = 22

                # Hysterese für Helligkeit: Wenn
                [[[[[enter_hysterese]]]]]
                    # ... wir bereits in "Nachführen" sind
                    as_value_laststate = var:current.state_id
                    # .... das Flag "Helligkeit > 25kLux" gesetzt ist
                    as_value_brightnessGt25k = true
                    # ... die Sonnenhöhe mindestens 18° ist
                    as_min_sun_altitude = 18
                    # ... die Sonne aus Richtung 130° bis 270° kommt
                    as_min_sun_azimut = 130
                    as_max_sun_azimut = 270
                    # Anmerkung: Hier keine erneute Prüfung der Temperatur, damit Temperaturschwankungen nicht
                    # zum Auf-/Abfahren der Raffstores führen

                # Verzögerter Ausstieg nach Unterschreitung der Mindesthelligkeit: Wenn
                [[[[[enter_delay]]]]]
                    # ... wir bereits in "Nachführen" sind
                    as_value_laststate = var:current.state_id
                    # .... das Flag "Helligkeit > 25kLux" nicht (!) gesetzt ist, aber diese Änderung nicht mehr als 20 Minuten her ist
                    as_value_brightnessGt25k = false
                    as_maxage_brightnessGt25k = 1200
                    # ... die Sonnenhöhe mindestens 18° ist
                    as_min_sun_altitude = 18
                    # ... die Sonne aus Richtung 130° bis 270° kommt
                    as_min_sun_azimut = 130
                    as_max_sun_azimut = 270
                    # Anmerkung: Auch hier keine erneute Prüfung der Temperatur, damit Temperaturschwankungen nicht
                    # zum Auf-/Abfahren der Raffstores führen

            # Zustand "Nachführen der Lamellen zum Sonnenstand bei großer Helligkeit", Gebäudeseite 2
            [[[[Nachfuehren_Seite_Zwei]]]]                      
                type = foo
                # Einstellungen des Vorgabezustands "Nachfuehren_Seite_Eins" übernehmen
                as_use = beispiel.default.raffstore.Nachfuehren_Seite_Eins

                # Sonnenwinkel in den Bedingungsgruppen anpassen
                [[[[[enter]]]]]
                    # ... die Sonne aus Richtung 220° bis 340° kommt
                    as_min_sun_azimut = 220
                    as_max_sun_azimut = 340

                [[[[[enter_hysterese]]]]]
                    # ... die Sonne aus Richtung 220° bis 340° kommt
                    as_min_sun_azimut = 220
                    as_max_sun_azimut = 340

                [[[[[enter_delay]]]]]
                    # ... die Sonne aus Richtung 220° bis 340° kommt
                    as_min_sun_azimut = 220
                    as_max_sun_azimut = 340                

            # Zustand "Nacht"
            [[[[Nacht]]]]
                type = foo
                name = Nacht
                # Aktionen:
                # - Behang ganz herunterfahren
                as_set_hoehe = value:100
                # - Lamellen ganz schließen
                as_set_lamelle = value:0
                # - "Suspend"-Item ggf. zurücksetzen               
                as_set_suspend = False

                # Einstieg in "Nacht": Wenn 
                [[[[[enter]]]]]
                    # - es zwischen 16:00 und 08:00 Uhr ist
                    as_min_time = 08:00
                    as_max_time = 16:00
                    as_negate_time = True
                    # - die Helligkeit höchstens 90 Lux beträgt
                    as_max_brightness = 90

            # Zustand "Morgens"
            [[[[Morgens]]]]
                type = foo
                name = Dämmerung Morgens
                # Aktionen:
                # - Behang ganz herunterfahren
                as_set_hoehe = value:100
                # - Lamellen ca 45° nach unten
                as_set_lamelle = value:25
                # - "Suspend"-Item ggf. zurücksetzen               
                as_set_suspend = False

                # Einstieg in "Morgens": Wenn 
                [[[[[enter]]]]]
                    # - die Helligkeit zwischen 90 und 250 Lux beträgt
                    as_min_brightness = 90
                    as_max_brightness = 250
                    # - es zwischen 00:00 und 12:00 Uhr ist
                    as_min_time = 00:00
                    as_max_time = 12:00

            # Zustand "Abends"
            [[[[Abends]]]]
                type = foo
                name = Dämmerung Abends
                # Aktionen:
                # - Behang ganz herunterfahren
                as_set_hoehe = value:100
                # - Lamellen ca 45° nach oben
                as_set_lamelle = value:75
                # - "Suspend"-Item ggf. zurücksetzen               
                as_set_suspend = False

                # Einstieg in "Abends": Wenn 
                [[[[[enter]]]]]
                    # - die Helligkeit zwischen 90 und 250 Lux beträgt
                    as_min_brightness = 90
                    as_max_brightness = 250
                    # - es zwischen 12:00 und 24:00 Uhr ist
                    as_min_time = 12:00
                    as_max_time = 24:00
                    # Anmerkung: "24:00" ist eigentlich eine ungültige Zeit. Sie wird aber automatisch zu 23:59:59 umgewandelt

            # Zustand "Tag"   
            [[[[Tag]]]]
                type = foo
                name = Tag (statisch)
                # Aktionen:
                # - Behang ganz hochfahren
                as_set_hoehe = value:0
                # - Lamellen auf den Standardwert bei ganz hochgefahrenem Behang
                as_set_lamelle = value:100
                # - "Suspend"-Item ggf. zurücksetzen               
                as_set_suspend = False

                # Einstieg in "Tag": Wenn 
                [[[[[enter]]]]]
                    # - es zwischen 06:30 und 21:30 Uhr ist
                    as_min_time = 06:30
                    as_max_time = 21:30
Automatisierung Raffstore 1

Jetzt wollen wir den ersten Raffstore automatisieren. Einige Items dazu haben wir sowieso schon, da der Raffstore über diese Items gesteuert wird.

[beispiel]
    [[raffstore1]]
        name = Raffstore Beispiel 1

        [[[aufab]]]
            type = bool
            name = Raffstore auf/ab fahren
            enforce_updates = on

        [[[step]]]
            type = bool
            name = Raffstore Schritt fahren/stoppen
            enforce_updates = on

        [[[hoehe]]]
            type = num
            name = Behanghöhe des Raffstores

        [[[lamelle]]]
            type = num
            name = Lamellenwinkel des Raffstores
Jetzt kommen noch die Items zur Automatisierung und schließlich das AutoBlind Objekt-Item hinzu:

[beispiel]
    [[raffstore1]]
        [[[automatik]]]
            [[[[lock]]]]
                type = bool
                name = Sperr-Item
                visu_acl = rw
                cache = on

            [[[[suspend]]]]
                type = bool
                name = Suspend-Item
                visu_acl = rw
                # Achtung: Beim "Suspend"-Item niemals "enforce_updates = yes" setzen! Das führt dazu dass das Setzen des
                # Suspend-Items bei der Initialisierung zu einem endlosen sofortigen Wiederaufruf der Statusermittlung führt!

            [[[[state_id]]]]
                type = str
                name = Id des aktuellen Zustands
                visu_acl = r
                cache = on

            [[[[state_name]]]]
                type = str
                name = Name des aktuellen Zustands
                visu_acl = r
                cache = on

            [[[[manuell]]]]
                type = bool
                name = Manuelle Bedienung
                # Änderungen dieser Items sollen als manuelle Bedienung gewertet werden
                eval_trigger = beispiel.raffstore1.aufab | beispiel.raffstore1.step | beispiel.raffstore1.hoehe | beispiel.raffstore1.lamelle
                # Änderungen, die ursprünglich von diesen Triggern (<caller>:<source>) ausgelöst wurden, sollen nicht als manuelle Bedienung gewertet werden
                as_manual_exclude = KNX:y.y.y | Init:*

            [[[[rules]]]]
                type = bool
                name = Automatik Raffstore 2
                as_plugin = active
                # Erste Zustandsermittlung nach 30 Sekunden
                as_startup_delay = 30
                # Über diese Items soll die Statusermittlung ausgelöst werden
                eval_trigger = beispiel.trigger.raffstore | beispiel.raffstore1.automatik.anwesenheit | beispiel.raffstore1.automatik.manuell | beispiel.raffstore1.automatik.lock | beispiel.raffstore1.automatik.suspend
                # In dieses Item soll die Id des aktuellen Zustands geschrieben werden
                as_laststate_item_id = ..state_id
                # In dieses Item soll der Name des aktuellen Zustands geschrieben werden
                as_laststate_item_name = ..state_name

                [[[[[Lock]]]]]
                    # Zustand "Lock": Nur die Vorgabeeinstellungen übernehmen
                    as_use = beispiel.default.raffstore.Lock

                [[[[[Suspend]]]]]
                    # Zustand "Suspend": Nur die Vorgabeeinstellungen übernehmen
                    as_use = beispiel.default.raffstore.Suspend

                [[[[[Nachfuehren]]]]]
                    # Zustand "Nachfuehren": Nur die Vorgabeeinstellungen übernehmen (Gebäudeseite 2)
                    as_use = beispiel.default.raffstore.Nachfuehren_Seite_Zwei

                [[[[[Nacht]]]]]
                    # Zustand "Nacht": Nur die Vorgabeeinstellungen übernehmen
                    as_use = beispiel.default.raffstore.Nacht

                [[[[[Morgens]]]]]
                    # Zustand "Morgens": Nur die Vorgabeeinstellungen übernehmen
                    as_use = beispiel.default.raffstore.Morgens

                [[[[[Abends]]]]]
                    # Zustand "Abends": Nur die Vorgabeeinstellungen übernehmen
                    as_use = beispiel.default.raffstore.Abends

                [[[[[Tag]]]]]
                    # Zustand "Tag": Vorgabeeinstellungen übernehmen
                    as_use = beispiel.default.raffstore.Tag
Automatisierung Raffstore 2

Der zweite Raffstore ist ein komplexeres Beispiel. Hier werden nicht nur die Vorgabewerte übernommen, hier werden komplett neue Bedingungsgruppen definiert, sowie vorhandene Bedingungsgruppen abgeändert.

[beispiel]
    [[raffstore2]]
        name = Raffstore Beispiel 2

        [[[aufab]]]
            type = bool
            name = Raffstore auf/ab fahren
            enforce_updates = on

        [[[step]]]
            type = bool
            name = Raffstore Schritt fahren/stoppen
            enforce_updates = on

        [[[hoehe]]]
            type = num
            name = Behanghöhe des Raffstores

        [[[lamelle]]]
            type = num
            name = Lamellenwinkel des Raffstores

        [[[automatik]]]
            [[[[lock]]]]
                type = bool
                name = Sperr-Item
                visu_acl = rw
                cache = on

            [[[[suspend]]]]
                type = bool
                name = Suspend-Item
                visu_acl = rw
                # Achtung: Beim "Suspend"-Item niemals "enforce_updates = yes" setzen! Das führt dazu dass das Setzen des
                # Suspend-Items bei der Initialisierung zu einem endlosen sofortigen Wiederaufruf der Statusermittlung führt!

            [[[[state_id]]]]
                type = str
                name = Id des aktuellen Zustands
                visu_acl = r
                cache = on

            [[[[state_name]]]]
                type = str
                name = Name des aktuellen Zustands
                visu_acl = r
                cache = on

            [[[[manuell]]]]
                type = bool
                name = Manuelle Bedienung
                # Änderungen dieser Items sollen als manuelle Bedienung gewertet werden
                eval_trigger = beispiel.raffstore2.aufab | beispiel.raffstore2.step | beispiel.raffstore2.hoehe | beispiel.raffstore2.lamelle
                # Änderungen, die ursprünglich von diesen Triggern (<caller>:<source>) ausgelöst wurden, sollen nicht als manuelle Bedienung gewertet werden
                as_manual_exclude = KNX:y.y.y | Init:*

            [[[[anwesenheit]]]]
                type = bool
                name = Anwesenheit im Raum
                eval = or
                eval_trigger = beispiel.steckdosen | beispiel.licht

            [[[[rules]]]]
                type = bool
                name = Automatik Raffstore 2
                as_plugin = active
                # Erste Zustandsermittlung nach 30 Sekunden
                as_startup_delay = 30
                # Über diese Items soll die Statusermittlung ausgelöst werden
                eval_trigger = beispiel.trigger.raffstore | beispiel.raffstore2.automatik.anwesenheit | beispiel.raffstore2.automatik.manuell | beispiel.raffstore2.automatik.lock | beispiel.raffstore2.automatik.suspend
                # In dieses Item soll die Id des aktuellen Zustands geschrieben werden
                as_laststate_item_id = ..state_id
                # In dieses Item soll der Name des aktuellen Zustands geschrieben werden
                as_laststate_item_name = ..state_name
                # Dieses Item zeigt die Anwesenheit im Raum
                as_item_anwesend = ..anwesenheit
                # Item das anzeigt, ob die Helligkeit außen mehr als 35kLux beträgt
                as_item_brightnessGt35k = beispiel.wetterstation.helligkeit.gt35k
                # Item das anzeigt, ob die Helligkeit außen mehr als 20Lux beträgt
                as_item_brightnessGt20k = beispiel.wetterstation.helligkeit.gt20k

                [[[[[Lock]]]]]
                    # Zustand "Lock": Nur die Vorgabeeinstellungen übernehmen
                    as_use = beispiel.default.raffstore.Lock

                [[[[[Suspend]]]]]
                    # Zustand "Suspend": Nur die Vorgabeeinstellungen übernehmen
                    as_use = beispiel.default.raffstore.Suspend

                [[[[[Nachfuehren]]]]]
                    # Zustand "Nachführen": Vorgabeeinstellungen übernehmen                   
                    as_use = beispiel.default.raffstore.Nachfuehren_Seite_Eins

                    # ..und jetzt verändern wir das ganze, in dem wir abhängig vom "Anwesend"-Flag andere
                    # Grenzwerte für die Helligkeit setzen.

                    # Erst definieren wir mal zusätzliche Einstiegsbedingungen, die die neuen Grenzwerte beinhalten:
                    [[[[[[enter_anwesend]]]]]]
                        # Einstieg in "Nachführen" bei Anwesenheit: Wenn 
                        # - das Flag "Anwesenheit" gesetzt ist
                        as_value_anwesend = true
                        # - das Flag "Helligkeit > 35kLux" seit mindestens 60 Sekunden gesetzt ist (also 8k Lux früher als in "enter")
                        as_value_brightnessGt35k = true
                        as_minage_brightnessGt35k = 60
                        # - die Sonnenhöhe mindestens 15° ist (also 3° früher als in "enter")
                        as_min_sun_altitude = 15
                        # - die Sonne aus Richtung 110° bis 270° kommt (also 20° früher als in "enter"
                        as_min_sun_azimut = 110
                        as_max_sun_azimut = 270

                    [[[[[[enter_anwesend_hysterese]]]]]]
                        # Hysterese für Helligkeit bei Anwesenheit: Wenn
                        # - das Flag "Anwesenheit" gesetzt ist
                        as_value_anwesend = true
                        # ... wir bereits in "Nachführen" sind
                        as_value_laststate = var:current.state_id
                        # .... das Flag "Helligkeit > 20kLux" gesetzt ist (also 5 kLux früher als in "enter_hysterese")
                        as_value_brightnessGt20k = true
                        # ... die Sonnenhöhe mindestens 15° ist (Übernahme aus "enter_anwesend")
                        as_min_sun_altitude = 15
                        # ... die Sonne aus Richtung 110° bis 270° kommt (Übernahme aus "enter_anwesend")
                        as_min_sun_azimut = 110
                        as_max_sun_azimut = 270

                    [[[[[[enter_anwesend_delay]]]]]]
                        # Verzögerter Ausstieg nach Unterschreitung der Mindesthelligkeit bei Anwesenheit: Wenn
                        # - das Flag "Anwesenheit" gesetzt ist
                        as_value_anwesend = true
                        # ... wir bereits in "Nachführen" sind
                        as_value_laststate = var:current.state_id
                        # .... das Flag "Helligkeit > 20kLux" nicht (!) gesetzt ist, aber diese Änderung nicht mehr als 20 Minuten her ist
                        as_value_brightnessGt20k = false
                        as_maxage_brightnessGt20k = 1200
                        # ... die Sonnenhöhe mindestens 15° ist (Übernahme aus "enter_anwesend")
                        as_min_sun_altitude = 15
                        # ... die Sonne aus Richtung 110° bis 270° kommt (Übernahme aus "enter_anwesend")
                        as_min_sun_azimut = 110
                        as_max_sun_azimut = 270

                    # Jetzt müssen wir die vorhandenen Bedingungen noch erweitern (sie gelten ja nur noch, wenn "Anwesenheit" nicht gesetzt ist)                   
                    [[[[[[enter]]]]]]
                        # Einstieg in "Nachführen": Wenn zusätzlich
                        # - das Flag "Anwesenheit" nicht gesetzt ist
                        as_value_anwesend = false

                    [[[[[[enter_hysterese]]]]]]
                        # Hysterese für Helligkeit: Wenn zusätzlich
                        # - das Flag "Anwesenheit" nicht gesetzt ist
                        as_value_anwesend = false
                    [[[[[[enter_delay]]]]]]
                        # Verzögerter Ausstieg nach Unterschreitung der Mindesthelligkeit:  Wenn zusätzlich
                        # - das Flag "Anwesenheit" nicht gesetzt ist
                        as_value_anwesend = false

                [[[[[Nacht]]]]]
                    # Zustand "Nacht": Vorgabeeinstellungen übernehmen
                    as_use = beispiel.default.raffstore.Nacht
                    # .. und zwei weitere Einstiegsbedingungen definieren

                    [[[[[[enter_schlafenszeit_woche]]]]]]
                        # Einstieg in "Nacht": Wenn 
                        # - es zwischen 21:00 und 07:00 Uhr ist
                        as_min_time = 07:00
                        as_max_time = 21:00
                        as_negate_time = True
                        # - der Wochentag zwischen Montag und Freitag liegt
                        as_min_weekday = 0
                        as_max_weekday = 4

                    [[[[[[enter_schlafenszeit_wochenende]]]]]]
                        # Einstieg in "Nacht": Wenn 
                        # - es zwischen 21:00 und 08:30 Uhr ist
                        as_min_time = 08:30
                        as_max_time = 21:00
                        as_negate_time = True
                        # - der Wochentag Samstag oder Sonntag ist
                        as_value_weekday = 5 | 6

                [[[[[Morgens]]]]]
                    # Zustand "Morgens": Nur die Vorgabeeinstellungen übernehmen
                    as_use = beispiel.default.raffstore.Morgens

                [[[[[Abends]]]]]
                    # Zustand "Abends": Nur die Vorgabeeinstellungen übernehmen
                    as_use = beispiel.default.raffstore.Abends

                [[[[[Tag]]]]]
                    # Zustand "Tag": Vorgabeeinstellungen übernehmen
                    as_use = beispiel.default.raffstore.Tag
