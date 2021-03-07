# vind_overlap_tijden
Een klein programma waarin medewerkers hun afwezigheden kunnen opgeven. Het programma moet vervolgens kunnen tonen of afwezigheidsperiodes elkaar overlappen.

# gebruikte tools
Python 3.8
Django 3.0.6

| Tables        | Are           | Cool  |
| ------------- |:-------------:| -----:|
| col 3 is      | right-aligned | $1600 |
| col 2 is      | centered      |   $12 |
| zebra stripes | are neat      |    $1 |

# Definitie datamodel

```python
from django.db import models

from claus_personen.models import Personen


class Aanwezigheid(models.Model):

      status = (('Actief','Actief'), ('Inactief','Inactief'))

      persoon = models.ForeignKey(Personen, on_delete=models.CASCADE)
    
      
      datum = models.DateField()
      begintijd = models.TimeField()
      eindtijd = models.TimeField()
      status = models.CharField(max_length=100, choices=status)

      class Meta:
        verbose_name_plural = "Aanwezigheid"
```
| id        | datum           | begintijd  | eindtijd | status| persoon_id
| ------------- |:-------------:| -----:|-----:|-----:|-----:|
| 22|2021-03-08|09:00:00|10:00:00|Actief|2 |
| 24|2021-03-07|09:00:00|10:00:00|Actief|1 |
| 25|2021-03-07|09:30:00|11:00:00|Actief|2 |



