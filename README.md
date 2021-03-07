# Vind overlappende tijden
Een klein programma waarin medewerkers hun afwezigheden kunnen opgeven. Het programma toont vervolgens of afwezigheidsperiodes elkaar overlappen.

# Visuele weergave

Een strokenplanning m.b.v. Google Charts laat zien aan de gebruiker waar de tijden elkaar raken.

<img src="https://github.com/C-Claus/vind_overlap_tijden/blob/main/00_overzicht.PNG" alt="drawing" width="800"/>

Door de begintijd en eindtijd te markeren met rode standlijnen is inzichtelijk hoe deze weergave zich verhoudt tot het datamodel

<img src="https://github.com/C-Claus/vind_overlap_tijden/blob/main/01_overzicht.PNG" alt="drawing" width="800"/>

Door de overlap te arceren kan het iets duidelijker worden waar de overlappingen zich bevinden

<img src="https://github.com/C-Claus/vind_overlap_tijden/blob/main/02_overzicht.PNG" alt="drawing" width="800"/>

# Gebruikte tools
* Python 3.8
  * datetime
* Django 3.0.6
  * crispy_forms
  * tables2
  * Google Charts
  



# models.py (Definitie datamodel)
Er zijn twee datamodellen gedefinieerd voor dit programma, "Personen" en "Aanwezigheid"

Het ```Personen``` datamodel

```python
class Personen(models.Model):

    in_dienst_status = (('Ja','Ja'),('Nee','Nee'),)
    goedkeurder_status  = (('Ja','Ja'),('Nee','Nee'),)

    persoonnr = models.CharField(max_length=30)
    naam = models.CharField(max_length=100)
    administratie_werkgever = models.ForeignKey(BedrijfsAdministratie, on_delete=models.CASCADE)
    in_dienst_status = models.CharField(max_length=200, choices=in_dienst_status)

    gebruikersgroep  = models.CharField(max_length=200, choices=gebruikersgroep_status)
 
    #User model komt uit de standaard Django tabel
    account = models.ForeignKey(User, on_delete=models.CASCADE)

    class Meta:
        verbose_name_plural = "Personen"

    def __str__(self):
        return self.naam
```

Het ```Aanwezigheid``` datamodel

```python
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

De volgende records uit het ```Aanwezigheid``` datamodel zullen gebruikt worden als voorbeeld om overlappende tijden te vinden.


| id        | datum           | begintijd  | eindtijd | status| persoon_id
| ------------- |:-------------:| -----:|-----:|-----:|-----:|
| 22|2021-03-08|09:00:00|10:00:00|Actief|2 |
| 24|2021-03-07|09:00:00|10:00:00|Actief|1 |
| 25|2021-03-07|09:30:00|11:00:00|Actief|2 |


# forms.py 

Met een Model based form kan snel een HTML formulier gedefinieerd worden met bootstrap stijl

```python
class DateInput(forms.DateInput):
    input_type = 'date'

class TimeInput(forms.TimeInput):
    input_type = 'time'

class AanwezigheidsForm(ModelForm):

    class Meta:
        model = Aanwezigheid
        fields = [  
                    
                    'datum',
                    'begintijd',
                    'eindtijd',
                    'status',
                    'persoon',
                    ]

        widgets = {'datum':DateInput(), 'begintijd':TimeInput(),'eindtijd':TimeInput()}  
```

# views.py

In views.py kunnen de bewerkingen gedaan en vind de interactie met gebruiker plaats.

```python
def aanwezigheid(response, persoon):

    persoon = Personen.objects.get(id=persoon)
    persoon_id = Personen.objects.filter(account_id=response.user.id).values_list('id', flat=True)[0]
    
    #haalt het formulier op uit forms.py
    aanwezigheid_form = AanwezigheidsForm(initial={'persoon': persoon,  'status':'Actief'})
    aanwezigheid_form.fields['status'].widget = forms.HiddenInput()
    aanwezigheid_form.fields['persoon'].widget = forms.HiddenInput()

    if response.method == 'POST':
        
        if 'save' in response.POST:
            aanwezigheid_form = AanwezigheidsForm(response.POST)
            aanwezigheid_form.save()

            #https://docs.djangoproject.com/en/3.1/ref/forms/api/#django.forms.Form.cleaned_data
            cleaned_data = aanwezigheid_form.cleaned_data
            aanwezigheid_datum =  Aanwezigheid()
            aanwezigheid_datum.datum = cleaned_data['datum']

            jaar = aanwezigheid_datum.datum.year
            maand = aanwezigheid_datum.datum.month
            dag = aanwezigheid_datum.datum.day

            url = reverse('aanwezigheids_overzicht', kwargs={ 'persoon_id': persoon_id,  'jaar': jaar, 'maand':maand, 'dag':dag })
            return HttpResponseRedirect(url)
            

    return render(response, "aanwezigheid/aanwezigheid.html", {"aanwezigheid_form":aanwezigheid_form,
                                                               "persoon":persoon}
                                                               )
```
# templates (map met HTML bestanden)

Templates is een map waarin de variabelen uit views.py gerenderd kunnen worden en response.POST kunnen worden teruggestuurd naar de views.py

```html
{% include "claus_portaal/base.html" %}
{% load crispy_forms_tags %}
{% block content %}
{% if user.is_authenticated %}

<form method='POST'>
  {% csrf_token %}
  {{ aanwezigheid_form | crispy }}
  <hr>
  <button type="submit" name="save" class="btn btn-outline-secondary btn-block">Registreer afwezigheid</button>
</form>


{% endif %}
{% endblock %}
```

# urls.py

In de urls.py worden de views aangeroepen, hier kunnen de diverse urls ook worden naamgegeven

```python
urlpatterns =   [   path('aanwezigheid/<int:persoon>', views.aanwezigheid, name="aanwezigheid"),
                    path('aanwezigheids_overzicht/<int:persoon_id>/<int:jaar>/<int:maand>/<int:dag>', views.aanwezigheids_overzicht, name="aanwezigheids_overzicht"),
                    path('aanwezigheids_overzicht_per_dag', views.aanwezigheid_dagoverzicht, name="aanwezigheid_dagoverzicht"),
                 ]
```                    

# visualisatie 

```html http://127.0.0.1:8000/claus_portaal ```

<img src="https://github.com/C-Claus/vind_overlap_tijden/blob/main/claus_portaal.png" alt="drawing" width="300"/>


```html http://127.0.0.1:8000/aanwezigheid/3 3 is in urls.py <int:persoon_id> ```

<img src="https://github.com/C-Claus/vind_overlap_tijden/blob/main/anwezigheid_slash_persoon_id.png" alt="drawing" width="300"/>


```html http://127.0.0.1:8000/aanwezigheids_overzicht_per_dag ```

<img src="https://github.com/C-Claus/vind_overlap_tijden/blob/main/aanwezigheids_overzicht_per_dag.png" alt="drawing" width="300"/>


# Benadering 

Er zijn meerdere manieren mogelijk om de raakvlakken te vinden, mijn idee is om een minuut_standlijn per minuut te laten itereren door de dag.
Als de minuut_standlijn zich bevind in een tijddelta van een gebruiker.
Een dictionary is hier het meest geschikt voor waarbij: key=persoon, value=tijddelta_minuten


in pseudocode:
```python
uit het datamodel komt het volgende:
"Harm": ["begintijd, 9:00", "eindtijd: 10:00"],
"Thijs": ["begintijd, 9:30", "eindtijd: 11:00"],
"Coen": ["begintijd, 9:15", "eindtijd: 10:00"]
            
            
afwezigheid_dict = {
            "Harm":['9:00','9:01',...','9:59''10:00]
            "Thijs":['9:30,,'9:31',....,',10:59,'11:00']]
            "Coen": ['9:15,''9:16',...','9:59', '10:00']]
             }
```

Door de overlap te arceren kan het iets duidelijker worden waar de overlappingen zich bevinden

<img src="https://github.com/C-Claus/vind_overlap_tijden/blob/main/02_overzicht.PNG" alt="drawing" width="800"/>

```python
##################################################
### 1 maak een list van standlijnen per minuut ###
##################################################

#aanname dat een werkdag loopt van 8:00 tot 18:00
#integer  480 = 8:00
#integer  1080 = 18:00

#integer 0 = 0:00
#integer 1440 = 23:59

#https://stackoverflow.com/questions/39298054/generating-15-minute-time-interval-array-in-python

datum_list = [datetime.datetime(jaar, maand, dag) + datetime.timedelta(minutes=1*x) for x in range(479, 1081)]    
minuten_list = [x.strftime('%Y-%m-%d %H:%M ') for x in datum_list]
```    

De output van ```python minuten ``` list

```python
2021-03-07 07:59 
2021-03-07 08:00
2021-03-07 08:01
2021-03-07 08:02
2021-03-07 08:03
..
..
2021-03-07 17:54
2021-03-07 17:55
2021-03-07 17:56
2021-03-07 17:57
2021-03-07 17:58
2021-03-07 17:59
2021-03-07 18:00
```

De volgende stap is om van alle gebruikers die hun afwezigheid hebben opgegeven voor die dag een tijddelta Δt
* Δt Harm = [eindtijd-begintijd] 
* Δt Thijs = [eindtijd-begintijd]
* Δt Coen = [eindtijd-begintijd]

Waarbij de waarde van [eindtijd-begintijd] een list is in minuten

```python
##########################################################
### 2 defineer een verzameling  van tijddeltas per key ###
##########################################################

persoon_minuten_delta_list  = []
vandaag  = datetime.datetime(jaar, maand, dag)

for i in afwezigheid_per_dag:
    start = date_time.combine(vandaag, i.begintijd)
    seconds = (date_time.combine(vandaag, i.eindtijd) - date_time.combine(vandaag, i.begintijd)).total_seconds()

    step = timedelta(minutes=1)

    for j in range(0, int(seconds)+60, int(step.total_seconds())):
        persoon_minuten_delta_list.append([str(i.persoon) + " begintijd:" + str(i.begintijd) + " eindtijd:" +  str(i.eindtijd) ,  (start) + (timedelta(seconds=j))])
```

Output persoon_minuten_delta_list

```python

['Harm  begintijd:09:00:00 eindtijd:10:00:00', datetime.datetime(2021, 3, 7, 9, 0)]
['Harm  begintijd:09:00:00 eindtijd:10:00:00', datetime.datetime(2021, 3, 7, 9, 1)]
['Harm  begintijd:09:00:00 eindtijd:10:00:00', datetime.datetime(2021, 3, 7, 9, 2)]
['Harm  begintijd:09:00:00 eindtijd:10:00:00', datetime.datetime(2021, 3, 7, 9, 3)]
['Harm  begintijd:09:00:00 eindtijd:10:00:00', datetime.datetime(2021, 3, 7, 9, 4)]
['Harm  begintijd:09:00:00 eindtijd:10:00:00', datetime.datetime(2021, 3, 7, 9, 5)]
....
'Harm  begintijd:09:00:00 eindtijd:10:00:00', datetime.datetime(2021, 3, 7, 9, 57)]
['Harm  begintijd:09:00:00 eindtijd:10:00:00', datetime.datetime(2021, 3, 7, 9, 58)]
['Harm  begintijd:09:00:00 eindtijd:10:00:00', datetime.datetime(2021, 3, 7, 9, 59)]
['Harm  begintijd:09:00:00 eindtijd:10:00:00', datetime.datetime(2021, 3, 7, 10, 0)]
['Thijs  begintijd:09:30:00 eindtijd:11:00:00', datetime.datetime(2021, 3, 7, 9, 30)]
['Thijs  begintijd:09:30:00 eindtijd:11:00:00', datetime.datetime(2021, 3, 7, 9, 31)]
['Thijs  begintijd:09:30:00 eindtijd:11:00:00', datetime.datetime(2021, 3, 7, 9, 32)]
...
['Thijs  begintijd:09:30:00 eindtijd:11:00:00', datetime.datetime(2021, 3, 7, 10, 58)]
['Thijs  begintijd:09:30:00 eindtijd:11:00:00', datetime.datetime(2021, 3, 7, 10, 59)]
['Thijs  begintijd:09:30:00 eindtijd:11:00:00', datetime.datetime(2021, 3, 7, 11, 0)]
['Coen  begintijd:09:15:00 eindtijd:10:30:00', datetime.datetime(2021, 3, 7, 9, 15)]
['Coen  begintijd:09:15:00 eindtijd:10:30:00', datetime.datetime(2021, 3, 7, 9, 16)]
['Coen  begintijd:09:15:00 eindtijd:10:30:00', datetime.datetime(2021, 3, 7, 9, 17)]
....
['Coen  begintijd:09:15:00 eindtijd:10:30:00', datetime.datetime(2021, 3, 7, 10, 28)]
['Coen  begintijd:09:15:00 eindtijd:10:30:00', datetime.datetime(2021, 3, 7, 10, 29)]
['Coen  begintijd:09:15:00 eindtijd:10:30:00', datetime.datetime(2021, 3, 7, 10, 30)]
```

De volgende stap is om te controleren in deze list waar de minuten overeenkomen per persoon
