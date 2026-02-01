# Guida: Creazione Skill "Mio Fantasma" (Alexa-Hosted)

Questa skill permette di identificare quale dispositivo Echo ha risposto e chi ha parlato (tramite `person_id`), inviando i dati a Home Assistant.

## 1. Creazione della Skill sulla Console Amazon

1. Accedi alla [Alexa Developer Console](https://developer.amazon.com/alexa/console/ask).
2. Clicca su **Create Skill**.
3. **Skill name:** `Mio Fantasma`.
4. **Primary locale:** `Italian (IT)`.
5. **Experience:** `Other` -> **Model:** `Custom`.
6. **Hosting service:** `Alexa-hosted (Python)`.
7. **Hosting region:** `EU (Ireland)`.
8. Scegli il template: **Start from Scratch**.

## 2. Modello di Interazione (Build)

1. Nella colonna di sinistra, vai su **Interaction Model** -> **JSON Editor**.
2. Sostituisci il contenuto con il seguente JSON:

```json
{
    "interactionModel": {
        "languageModel": {
            "invocationName": "gestore automazioni vocali",
            "intents": [
                { "name": "AMAZON.CancelIntent", "samples": [] },
                { "name": "AMAZON.HelpIntent", "samples": [] },
                { "name": "AMAZON.StopIntent", "samples": [] },
                { "name": "AMAZON.NavigateHomeIntent", "samples": [] },
                { "name": "AMAZON.FallbackIntent", "samples": [] },
                {
                    "name": "EseguiProceduraIntent",
                    "slots": [
                        {
                            "name": "procedura",
                            "type": "AMAZON.SearchQuery"
                        }
                    ],
                    "samples": [
                        "esegui {procedura}",
                        "avvia procedura {procedura}",
                        "attivare {procedura}",
                        "di attivare {procedura}"
                    ]
                }
            ],
            "types": []
        }
    }
}

```

3. Clicca **Save Model** e poi **Build Model**.

## 3. Configurazione del Codice (Code)

Sposta la navigazione sulla scheda **Code** in alto.

### File: `requirements.txt`

```text
ask-sdk-core==1.19.0
urllib3<2.0.0

```

### File: `config.json`

Crea un nuovo file (tasto destro sulla cartella `lambda` -> **New File**) nominato `config.json`:

```json
{
  "url": "https://TUO_INDIRIZZO_HA/api",
  "bearer_token": "IL_TUO_TOKEN_LUNGA_DURATA",
  "speech_confirmation_message": "<speak>Attivo <nome_procedura><break strength='medium'/>.</speak>"
}

```

* **url:** L'indirizzo pubblico di Home Assistant.
* **speech_confirmation_message:** La frase che Alexa pronuncerà alla chiusura della skill.

### File: `lambda_function.py`

Sostituisci tutto il codice con questa versione che legge la risposta dal file di configurazione:

```python
import json
import logging
import os
import urllib3
from datetime import datetime, timezone
from ask_sdk_core.skill_builder import SkillBuilder
from ask_sdk_core.dispatch_components import AbstractRequestHandler, AbstractExceptionHandler
from ask_sdk_core.utils import is_request_type, is_intent_name

logger = logging.getLogger()
logger.setLevel(logging.INFO)
http = urllib3.PoolManager()

def get_config():
    path = os.path.join(os.path.dirname(__file__), 'config.json')
    with open(path, 'r') as f:
        return json.load(f)

config = get_config()

class GhostTrackerHandler(AbstractRequestHandler):
    def can_handle(self, handler_input):
        return (is_request_type("LaunchRequest")(handler_input) or 
                is_intent_name("EseguiProceduraIntent")(handler_input))

    def handle(self, handler_input):
        intent = getattr(handler_input.request_envelope.request, 'intent', None)
        nome_procedura = "default"
        if intent and 'procedura' in intent.slots and intent.slots['procedura'].value:
            nome_procedura = intent.slots['procedura'].value

        system = handler_input.request_envelope.context.system
        device_id = system.device.device_id
        person_id = system.person.person_id if hasattr(system, 'person') and system.person else None
        final_identity = person_id if person_id else system.user.user_id
        id_type = "VOCAL_ID" if person_id else "ACCOUNT_ID"

        self._update_ha(device_id, final_identity, id_type, nome_procedura)

        raw_speech = config.get('speech_confirmation_message', 'Attivo <nome_procedura>')
        procedura_testo = nome_procedura.replace("_", " ")
        speech_text = raw_speech.replace("<nome_procedura>", procedura_testo)

        return (handler_input.response_builder.speak(speech_text)
                .set_should_end_session(True).response)

    def _update_ha(self, dev_id, identity, id_type, proc):
        now = datetime.now(timezone.utc) 
        url = config.get('url', '').rstrip('/')
        api_url = f"{url}/states/input_text.alexa_last_called_device"
        
        payload = {
            "state": dev_id,
            "attributes": {
                "person_id": identity,
                "id_type": id_type,
                "procedura": proc,
                "timestamp": now.isoformat(),
                "friendly_name": "Gestore Automazioni Vocali (Last Called)"
            }
        }
        headers = {"Authorization": f"Bearer {config.get('bearer_token')}", "Content-Type": "application/json"}
        try:
            encoded_data = json.dumps(payload).encode('utf-8')
            http.request('POST', api_url, body=encoded_data, headers=headers, timeout=2.0)
        except Exception as e:
            logger.error(f"Errore invio a HA: {e}")

sb = SkillBuilder()
sb.add_request_handler(GhostTrackerHandler())
lambda_handler = sb.lambda_handler()

```

*Clicca **Save** e poi **Deploy**.*

## 4. Abilitazione Identità Vocale (Permessi)

Questo passaggio è fondamentale per ricevere il `person_id`:

1. **Console Web:** Vai su **Tools** -> **Permissions** e attiva **Skill Personalization**. Torna nel **JSON Editor** e clicca di nuovo su **Build Model**.
2. **App Smartphone:** Apri l'app Alexa -> *Altro* -> *Skill e Giochi* -> *Le tue Skill* -> *Sviluppatore* -> *Mio Fantasma*. Clicca su **Impostazioni** -> **Autorizzazioni Skill** e attiva **Personalizzazione della skill**.

## 5. Configurazione Home Assistant

Assicurati che l'entità di destinazione esista nel tuo sistema:

```yaml
# Esempio in configuration.yaml
input_text:
  alexa_last_called_device:
    name: "Alexa Last Called Device"

```



### Automazione e Trigger (Pyscript)

Il sistema reagisce istantaneamente grazie al trigger definito in un pyscript:

```python
@state_trigger("True", watch=["input_text.alexa_last_called_device.procedura","input_text.alexa_last_called_device.timestamp"])
def dispatcher_procedure_vocali(value=None, **kwargs):
    """
    Questo trigger monitora l'attributo 'procedura' e il 'timestamp'.
    Lancia automaticamente la procedura indicata negli attributi dell'helper.
    """

```

oppure con automazione HA equivalente
In questo esempio definiamo come procedura "riscaldamento"

### Automazione HA: Dispatcher Procedure Vocali

```yaml
alias: "Alexa: Dispatcher Procedure Vocali"
description: "Alternativa al trigger PyScript per lanciare procedure basate sugli attributi dell'helper"
mode: parallel # Consente esecuzioni simultanee se arrivano comandi rapidi

trigger:
  - platform: state
    entity_id: input_text.alexa_last_called_device
    # Monitoriamo il timestamp perché cambia ad ogni chiamata, assicurando il trigger
    attribute: timestamp

condition:
  # 1. Verifica che la procedura non sia vuota o di default
  - condition: template
    value_template: >
      {% set proc = state_attr('input_text.alexa_last_called_device', 'procedura') %}
      {{ proc is not none and proc != 'default' }}

  # 2. Controllo Latenza: il comando deve essere stato inviato negli ultimi 10 secondi
  - condition: template
    value_template: >
      {% set ts = state_attr('input_text.alexa_last_called_device', 'timestamp') %}
      {{ (now() - as_datetime(ts)).total_seconds() < 10 if ts else false }}

action:
  - choose:
      # CASO: Riscaldamento
      - conditions:
          - condition: template
            value_template: "{{ state_attr('input_text.alexa_last_called_device', 'procedura') == 'riscaldamento' }}"
        sequence:
          # Chiamata alla funzione PyScript esistente
          - service: script.che_esegue_la_tua_procedura_solita_o_nuova_con_alexa_actions
            data: {}

      # Qui puoi aggiungere altri casi per nuove procedure (es. luci, allarme)
      # - conditions: ...
      #   sequence: ...

    default:
      - service: logbook.log
        data:
          name: "Alexa Dispatcher"
          message: "Procedura non riconosciuta o ignorata"

```

---

## 6. Utilizzo

Puoi interagire con il sistema in modo naturale:

* **Comando Diretto**: *"Alexa, chiedi a gestore automazioni vocali di attivare riscaldamento"*.
* **Invocazione Semplice**: *"Alexa, apri mio fantasma"*.

L'entità su HA verrà aggiornata con l'ID del dispositivo, l'ID vocale di chi ha parlato e la procedura richiesta. Il trigger `dispatcher_procedure_vocali` provvederà poi a smistare il comando alla funzione corretta.

Con il **Dispatcher** la chiamata alla skill stessa trasporta tutte le informazioni (chi, dove e cosa fare) necessarie a far scattare la logica in Home Assistant.

Ecco le istruzioni della metodologia:

---

## FASE 7: Integrazione nelle Routine Alexa (Automazione Integrata)

Grazie al nuovo **Dispatcher vocale** implementato su Home Assistant, la routine Alexa ora serve solo a inviare il "pacchetto dati" iniziale. Sarà poi il tuo server a leggere la procedura richiesta e ad avviare il dialogo corretto dall'Echo che ha risposto.

**Scenario di Esempio:** Vuoi dire *"Alexa, ho freddo"* e avviare il dialogo del riscaldamento sapendo chi sei e in che stanza ti trovi.

1. Apri l'app **Alexa** sul telefono e vai su **Altro** -> **Routine**.
2. **Crea una nuova routine** (+) o modificane una esistente.
3. **Quando:** Voce -> Inserisci la tua frase naturale, ad esempio: *"Alexa, ho freddo"* oppure *"Alexa, riscaldamento"*.
4. **Aggiungi un'azione ("ALEXA ESEGUIRÀ LE SEGUENTI AZIONI"):**
### A. Inserimento dell'Azione Custom (Personalizzata)


* Clicca su **Aggiungi un'azione**.
* Seleziona l'icona **Personalizzata** (⌨️).
* Nel campo di testo, scrivi esattamente il comando per la tua skill:
> `Chiedi a gestore automazioni vocali di attivare riscaldamento`


* Tocca **Avanti**.


### B. Selezione del Dispositivo di Risposta (Context-Awareness)


* Nella schermata di riepilogo della routine, sotto la voce **"Alexa risponderà da"**, è fondamentale selezionare **Il dispositivo a cui rispondi**.
* Questo garantisce che il `device_id` inviato a Home Assistant sia quello corretto della stanza in cui ti trovi.


5. Tocca **Salva** la Routine.
