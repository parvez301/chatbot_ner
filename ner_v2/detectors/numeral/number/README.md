## Number Detector 

This is the V2 version of Number detector module that will detect number and number word from text in multiple languages. This detector module adds an additional feature of detecting units along with number in text. Example -  for a given text "5 kg", this module will return `5`  as detected value and `kg` as detected unit. 

 We are currently providing number detection supports in 6 languages, which are

- English
- Hindi
- Marathi
- Gujarati
- Telgu
- Tamil

### Usage

- **Python Shell**

  ```python
  >> from ner_v2.detector.number.number.date_detection import NumberDetector
  >> detector = NumberDetector(entity_name='number', language='en')  # here language will be ISO 639-1 code
  >> detector.detect_entity(text= 'I want 4000 rs')
  >> {'entity_value': [{'value': '4000', 'unit': 'rupees'}], 'original_text':['4000 rs']}
  ```

- **Curl Command**

  ```bash
  # For a sample query with following parameters
  # message="do hajaar chaar sau"
  # entity_name='number'
  # structured_value=None
  # fallback_value=None
  # bot_message=None
  # min_number_digits=1
  # max_number_digits=6
  # source_language='hi'
  # language_script='hi'
  
  $ URL='localhost'
  $ PORT=8081
  
  $ curl -i 'http://'$URL':'$PORT'/v2/date?message=do%20hajaar%20char%20sau&entity_name=number&structured_value=&fallback_value=&bot_message=&min_number_digits=1&max_number_digits=6&source_language=hi&language_script=hi'
  
  # Curl output
  $ {
      "data": [
          {
              "detection": "message",
              "original_text": "do hajaar chaar sau",
              "entity_value": {
                  "unit": null,
                  "value": "2400"
              },
              "language": "hi"
          }
      ]
  }
  ```

  

### Steps to add new language for Number detection

In order to add any new language you have to follow below steps:

1. Create a directory with `ISO 639-1` code of that language inside `ner_v2/detectors/numeral/number/`.  

2. Create a directory named `data` inside language_code folder.

3. Add two files named `numerals_constant.csv`, `units.csv` inside data folder.  

   Below is the folder structure of same after adding all the files for new language `xy`.

   ```python
   |__ner_v2
         |___detectors
             |___numeral
                 |___number
                     |___xy    # <- New language Added 
                     |	  |___data
                     |      |___numerals_constant.csv
                     |      |___units.csv
                     |
                     |__number_detection.py 
   ```




####  GuideLines to create data files

Below is the brief about how to create data files `numerals_constant.csv`, `units.csv. `  All the description of each file is explained using hindi as a reference language. 

1. **numerals_constant.csv**:  This files contains the vocabs for number, their numerals/name variants, correspoding number value and their type .

   | number |     name_variants      | value  | type  |
   | :----: | :--------------------: | :----: | :---: |
   |   ०    |    शून्य\|zero\|sunya    |   0    | unit  |
   |   १    |       एक\|ek\|ik       |   1    | unit  |
   |  १.५   | डेढ़\|ढेड़\|देढ़\|dedh\|dhed |  1.5   | unit  |
   |  १००   |      सौ\|sau\|sao      |  100   | scale |
   | १००००० | लाख\|lakh\|laakh\|lac  | 100000 | scale |

   Here, 1st column will contain the number in their respective language script. 2nd column corresponds to numerals or name variants of number in respective language script and in english script.  

   3rd column contains the value representation of language number. 

   4th column define the whether the number is unit or scale. Units are individual numbers while scale are used with unit number to form numbers. Example - for number "200" , `2` is unit number and scale of  `100` is used to form a new number i.e `200`.    

2. **units.csv**:  This files contains the vocabs of units for number and their correspoding all variants possible .

   | unit_value |                        unit_variants                         |
   | :--------: | :----------------------------------------------------------: |
   |   rupees   | rupees \| rupay \| paisa \| paise \| inr \| रूपीस \| रुपया \| रूपए\| पैसा\| पैसे\| ₹ |
   |   dollar   |                  Dollar \| usd \| डॉलर \| $                  |

   Here, 1st column will contain the value of unit which will be return by detector module, while 2nd column contain all the possible variants of units in language and english script.

#### Guidelines to add new detectors for number apart from builtin ones:

Create a new class `number_detection.py`  inside language directory.
To start with copy the code below

```python
import re
import os

from ner_v2.constant import LANGUAGE_DATA_DIRECTORY
from ner_v2.detectors.numeral.constant import NUMBER_DETECT_UNIT, NUMBER_DETECT_VALUE
from ner_v2.detectors.numeral.number.standard_number_detector import BaseNumberDetector


class NumberDetector(BaseNumberDetector):
    data_directory_path = os.path.join((os.path.dirname(os.path.abspath(__file__)).rstrip(os.sep)),
                                       LANGUAGE_DATA_DIRECTORY)

    def __init__(self, entity_name):
        super(NumberDetector, self).__init__(entity_name=entity_name,                                      data_directory_path=NumberDetector.data_directory_path)
```

Note that the class name must be `NumberDetector` 
and should inherit from `ner_v2.detectors.numeral.number.standard_number_detector.BaseNumberDetector`

Next we define a custom detector. For our purposes we will add a detector to detect text '2 people' and `2` as number and unit as `people`.

1. The custom detector must accept two arguments `number_list` and `original_list` and must operate on `self.processed_text`
2. The two arguments `number_list` and `original_list` can both be either None or lists of same length.
   `number_list` contains parsed number values and `original_list` contains their corresponding text substrings in the passed text that were detected as numbers.
3. Your detector must appened parsed number dicts with keys `'value', 'unit'` to `number_list`
   and to `original_list` their corresponding substrings from `self.processed_text` that were parsed into numbers.
4. Take care to not mutate `self.processed_text` in any way as main detect method in base class depends on it to eliminate already detected number from it after each detector is run.
5. Finally your detector must return a tuple of (number_list, original_list). Ensure that `number_list` and `original_list` are of equal lengths before returning them.

```python
    def _custom_detect_number_of_people_format(self, number_list=None, original_list=None):
        number_list = number_list or []
        original_list = original_list or []
        patterns = re.findall(r'\s((fo?r)*\s*([0-9]+)\s*(ppl|people|passengers?|travellers?|persons?|pax|adults?))\s',
                              self.processed_text.lower())
        for pattern in patterns:
            number_list.append({
                NUMBER_DETECT_VALUE: pattern[2],
                NUMBER_DETECT_UNIT: 'people'
            })
            original_list.append(pattern[0])

        return number_list, original_list
```

Once having defined a custom detector, we now add it to `self.detector_preferences` attribute. You can simply append your custom detectors at the end of this list or you can copy the default ordering from 
`detectors.numeral.number.standard_number_detector.BaseNumberDetector` and inject your own detectors in between.
Below we show an example where we put our custom detector on top to execute it before some builtin detectors.

```python
	def __init__(self, entity_name):
        super(NumberDetector, self).__init__(entity_name=entity_name,
                                             data_directory_path=NumberDetector.data_directory_path)

        self.detector_preferences = [
            self._custom_detect_number_of_people_format,
            self._detect_number_from_digit,
            self._detect_number_from_numerals
        ]
```

Putting it all together, we have

```python
import re
import os

from ner_v2.constant import LANGUAGE_DATA_DIRECTORY
from ner_v2.detectors.numeral.constant import NUMBER_DETECT_UNIT, NUMBER_DETECT_VALUE
from ner_v2.detectors.numeral.number.standard_number_detector import BaseNumberDetector


class NumberDetector(BaseNumberDetector):
    data_directory_path = os.path.join((os.path.dirname(os.path.abspath(__file__)).rstrip(os.sep)),
                                       LANGUAGE_DATA_DIRECTORY)

    def __init__(self, entity_name):
        super(NumberDetector, self).__init__(entity_name=entity_name,
                                             data_directory_path=NumberDetector.data_directory_path)

        self.detector_preferences = [
            self._custom_detect_number_of_people_format,
            self._detect_number_from_digit,
            self._detect_number_from_numerals
        ]

    def _custom_detect_number_of_people_format(self, number_list=None, original_list=None):
        number_list = number_list or []
        original_list = original_list or []
        patterns = re.findall(r'\s((fo?r)*\s*([0-9]+)\s*(ppl|people|passengers?|travellers?|persons?|pax|adults?))\s',
                              self.processed_text.lower())
        for pattern in patterns:
            number_list.append({
                NUMBER_DETECT_VALUE: pattern[2],
                NUMBER_DETECT_UNIT: 'people'
            })
            original_list.append(pattern[0])

        return number_list, original_list

```

For a working example, please refer `ner_v2/detectors/numerals/number/en/date_detection.py`

**Please note that the API right now is too rigid and we plan to change it to make it much more
easier to extend in the future**
