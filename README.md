# mio-fantasma

Alexa skill that allows identifying which Echo device responded and who spoke (via `person_id`), sending the data to Home Assistant.

# 👻 Alexa Skill: "Mio Fantasma"

## What is "Mio Fantasma"?

**Mio Fantasma** (My Ghost) is a custom Alexa-hosted skill that acts as an intelligent bridge between Amazon Alexa and **Home Assistant**.

Unlike standard integrations, this skill acts as a real-time "context sensor". When activated, it transmits three specific and unique pieces of data to Home Assistant:

* **Who spoke:** Identifies the user via their `person_id` (Voice ID).
* **Where they are:** Captures the ID of the specific Echo device that received the command.
* **What they want to do:** Sends a specific "procedure" (e.g., "heating," "goodnight," etc.).

---

## Why did I create it?

The primary need arose to maximize the effectiveness of automations based on **[alexa-actions](https://github.com/keatontaylor/alexa-actions)**.

The main limitation of many integrations lies in the difficulty of starting an interactive dialogue on the correct device and with the correct person. "Mio Fantasma" solves this problem by ensuring:

1. **Device Certainty (Where):** The skill captures the ID of the Echo that responded to the routine. This allows `alexa-actions` to immediately "hook" the dialogue flow onto that specific device, preventing the question from being asked in the wrong room.
2. **Identity Certainty (Who):** Thanks to `person_id` recognition, Home Assistant knows who is initiating the action. This allows for personalizing the dialogue flow or restricting critical actions to authorized users only.
3. **Fluid Dialogue Flow:** The skill allows for the immediate start of the `alexa-actions` question-and-answer sequence, making the interaction natural and free of routing errors.

---

## Technical Notes and Optimization

* **Instant Trigger:** The skill does not execute actions directly; instead, it populates an `input_text` entity in Home Assistant. This state change is detected instantly (e.g., via PyScript), serving as a trigger for the subsequent logic.
* **Latency Management (User Experience):** By nature, triggering an automation via the cloud (Alexa -> Skill -> HA -> Alexa-Actions) can involve minor processing delays.
* **Response Message Trick:** You can visually and acoustically "mask" these latency times by slightly increasing the length of the skill's response message, which is configurable in the `config.json` file.
* This ensures that the `device_id` sent to Home Assistant is the correct one for the room you are currently in.

---

---

# mio-fantasma
Skill Alexa che permette di identificare quale dispositivo Echo ha risposto e chi ha parlato (tramite `person_id`), inviando i dati a Home Assistant

# 👻 Skill Alexa: "Mio Fantasma"

## Cos'è "Mio Fantasma"?

**Mio Fantasma** è una skill Alexa-hosted personalizzata che funge da ponte intelligente tra Amazon Alexa e **Home Assistant**.

A differenza delle integrazioni standard, questa skill agisce come un "sensore di contesto" in tempo reale. Quando viene attivata, trasmette a Home Assistant tre dati certi e univoci:

* **Chi ha parlato:** Identifica l'utente tramite il `person_id` (Identità Vocale).
* **Dove si trova:** Cattura l'ID del dispositivo Echo specifico che ha ricevuto il comando.
* **Cosa vuole fare:** Invia una "procedura" specifica (es. "riscaldamento", "buonanotte", ecc.).

## Perché l'ho creata?

L'esigenza principale è nata per massimizzare l'efficacia delle automazioni basate su **[alexa-actions](https://github.com/keatontaylor/alexa-actions)**.

Il limite principale di molte integrazioni risiede nella difficoltà di avviare un dialogo interattivo sul dispositivo corretto e con la persona corretta. "Mio Fantasma" risolve questo problema garantendo:

1. **Certezza del Dispositivo (Dove):** La skill cattura l'ID dell'Echo che ha risposto alla routine, permettendo ad `alexa-actions` di "agganciare" immediatamente il flusso del dialogo su quello specifico dispositivo, evitando che la domanda venga posta nella stanza sbagliata.
2. **Certezza dell'Identità (Chi):** Grazie al riconoscimento del `person_id`, Home Assistant sa chi sta avviando l'azione. Questo permette di personalizzare il flusso del dialogo o limitare azioni critiche solo a utenti autorizzati.
3. **Flusso del Dialogo Fluido:** La skill permette di avviare immediatamente la sequenza di domande e risposte di `alexa-actions`, rendendo l'interazione naturale e priva di errori di indirizzamento.

## Note Tecniche e Ottimizzazione

* **Trigger Istantaneo:** La skill non esegue azioni direttamente, ma popola un `input_text` su Home Assistant. Questo cambio di stato viene rilevato istantaneamente (es. tramite PyScript), fungendo da trigger per la logica successiva.
* **Gestione della Latenza (User Experience):** Per natura, l'innesco di un'automazione tramite cloud (Alexa -> Skill -> HA -> Alexa-Actions) può presentare dei piccoli ritardi di elaborazione.
* **Trucco del Messaggio di Risposta:** È possibile "mascherare" visivamente e acusticamente questi tempi di latenza aumentando leggermente la lunghezza del messaggio di risposta della skill (configurabile nel file `config.json`).


* Questo garantisce che il `device_id` inviato a Home Assistant sia quello corretto della stanza in cui ti trovi.

