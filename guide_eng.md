# Guide: Creating the "Mio Fantasma" Skill (Alexa-Hosted)

This skill allows identifying which Echo device responded and who spoke (via `person_id`), sending the data to Home Assistant.

## 1. Skill Creation on the Amazon Console

1. Access the [Alexa Developer Console](https://developer.amazon.com/alexa/console/ask).
2. Click on **Create Skill**.
3. **Skill name:** `Mio Fantasma`.
4. **Primary locale:** `Italian (IT)`.
5. **Experience:** `Other` -> **Model:** `Custom`.
6. **Hosting service:** `Alexa-hosted (Python)`.
7. **Hosting region:** `EU (Ireland)`.
8. Choose the template: **Start from Scratch**.

## 2. Interaction Model (Build)

1. In the left column, go to **Interaction Model** -> **JSON Editor**.
2. Replace the content with the following JSON:

```json
{
    "interactionModel": {
        "languageModel": {
            "invocationName": "voice automation manager",
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
                        "execute {procedura}",
                        "start procedure {procedura}",
                        "activate {procedura}",
                        "to activate {procedura}"
                    ]
                }
            ],
            "types": []
        }
    }
}

```

3. Click **Save Model** and then **Build Model**.

## 3. Code Configuration (Code)

Switch to the **Code** tab at the top.

### File: `requirements.txt`

```text
ask-sdk-core==1.19.0
urllib3<2.0.0

```

### File: `config.json`

Create a new file (right-click on the `lambda` folder -> **New File**) named `config.json`:

```json
{
  "url": "https://YOUR_HA_ADDRESS/api",
  "bearer_token": "YOUR_LONG_LIVED_TOKEN",
  "speech_confirmation_message": "<speak>Activating <nome_procedura><break strength='medium'/>.</speak>"
}

```

* **url:** The public address of Home Assistant.
* **speech_confirmation_message:** The phrase Alexa will speak when the skill closes.

### File: `lambda_function.py`

Replace all the code with this version that reads the response from the configuration file:

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

        # Read the confirmation message from config
        raw_speech = config.get('speech_confirmation_message', 'Activating <nome_procedura>')
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
                "friendly_name": "Voice Automation Manager (Last Called)"
            }
        }
        headers = {"Authorization": f"Bearer {config.get('bearer_token')}", "Content-Type": "application/json"}
        try:
            encoded_data = json.dumps(payload).encode('utf-8')
            http.request('POST', api_url, body=encoded_data, headers=headers, timeout=2.0)
        except Exception as e:
            logger.error(f"Error sending to HA: {e}")

sb = SkillBuilder()
sb.add_request_handler(GhostTrackerHandler())
lambda_handler = sb.lambda_handler()

```

*Click **Save** and then **Deploy**.*

## 4. Enabling Voice Identity (Permissions)

This step is critical to receive the `person_id`:

1. **Web Console:** Go to **Tools** -> **Permissions** and enable **Skill Personalization**. Go back to the **JSON Editor** and click **Build Model** again.
2. **Smartphone App:** Open the Alexa app -> *More* -> *Skills & Games* -> *Your Skills* -> *Dev* -> *Mio Fantasma*. Click **Settings** -> **Skill Permissions** and enable **Skill Personalization**.

## 5. Home Assistant Configuration

Ensure the target entity exists in your system:

```yaml
# Example in configuration.yaml
input_text:
  alexa_last_called_device:
    name: "Alexa Last Called Device"

```

### Automation and Trigger (Pyscript)

The system reacts instantly thanks to the trigger defined in a pyscript:

```python
@state_trigger("True", watch=["input_text.alexa_last_called_device.procedura","input_text.alexa_last_called_device.timestamp"])
def dispatcher_procedure_vocali(value=None, **kwargs):
    """
    This trigger monitors the 'procedura' attribute and the 'timestamp'.
    It automatically launches the procedure specified in the helper's attributes.
    """

```

or with an equivalent HA automation. In this example, we define "heating" as the procedure:

### HA Automation: Voice Procedure Dispatcher

```yaml
alias: "Alexa: Voice Procedure Dispatcher"
description: "Alternative to the PyScript trigger to launch procedures based on helper attributes"
mode: parallel # Allows simultaneous executions if rapid commands arrive

trigger:
  - platform: state
    entity_id: input_text.alexa_last_called_device
    # Monitor the timestamp because it changes with every call, ensuring the trigger fires
    attribute: timestamp

condition:
  # 1. Verify that the procedure is not empty or default
  - condition: template
    value_template: >
      {% set proc = state_attr('input_text.alexa_last_called_device', 'procedura') %}
      {{ proc is not none and proc != 'default' }}

  # 2. Latency Control: the command must have been sent within the last 10 seconds
  - condition: template
    value_template: >
      {% set ts = state_attr('input_text.alexa_last_called_device', 'timestamp') %}
      {{ (now() - as_datetime(ts)).total_seconds() < 10 if ts else false }}

action:
  - choose:
      # CASE: Heating
      - conditions:
          - condition: template
            value_template: "{{ state_attr('input_text.alexa_last_called_device', 'procedura') == 'riscaldamento' }}"
        sequence:
          # Call to the existing PyScript function or your usual procedure
          - service: script.your_procedure_script_with_alexa_actions
            data: {}

      # Add more cases for new procedures (e.g., lights, alarm) here
    default:
      - service: logbook.log
        data:
          name: "Alexa Dispatcher"
          message: "Procedure not recognized or ignored"

```

---

## 6. Usage

You can interact with the system naturally:

* **Direct Command**: *"Alexa, ask voice automation manager to activate heating"*.
* **Simple Invocation**: *"Alexa, open mio fantasma"*.

The entity in HA will be updated with the device ID, the voice ID of the speaker, and the requested procedure. The `dispatcher_procedure_vocali` will then route the command to the correct function.

---

## PHASE 7: Integration into Alexa Routines (Integrated Automation)

Thanks to the new **Voice Dispatcher** implemented on Home Assistant, the Alexa routine now only serves to send the initial "data packet". Your server will then read the requested procedure and start the correct dialogue from the Echo that responded.

**Example Scenario:** You want to say *"Alexa, I'm cold"* and start the heating dialogue knowing who you are and which room you are in.

1. Open the **Alexa app** on your phone and go to **More** -> **Routines**.
2. **Create a new routine** (+) or modify an existing one.
3. **When this happens:** Voice -> Enter your natural phrase, for example: *"Alexa, I'm cold"* or *"Alexa, heating"*.
4. **Add an action ("ALEXA WILL"):**

### A. Inserting the Custom Action

* Click **Add an action**.
* Select the **Custom** (⌨️) icon.
* In the text field, write exactly the command for your skill:
> `Ask voice automation manager to activate heating`


* Tap **Next**.

### B. Selecting the Responding Device (Context-Awareness)

* In the routine summary screen, under **"Alexa will respond from"**, it is essential to select **The device you speak to**.
* This ensures that the `device_id` sent to Home Assistant is the correct one for the room you are in.

5. Tap **Save** the Routine.
