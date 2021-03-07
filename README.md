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
| 24|2021-03-07|09:00:00|10:00:00|Actief|1 |
| 25|2021-03-07|09:30:00|11:00:00|Actief|2 |
| 25|2021-03-07|09:15:00|10:30:00|Actief|3 |



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

vandaag  = datetime.datetime(jaar, maand, dag)

    persoon_ingelogd_minuten_delta_list  = []
    persoon_overige_minuten_delta_list  = []

    persoon_ingelogd_dict = {}
    persoon_overige_dict = {}

    #loop over het datamodel
    for i in afwezigheid_per_dag:

        #initialiseer de minuten
        start = date_time.combine(vandaag, i.begintijd)
        seconds = (date_time.combine(vandaag, i.eindtijd) - date_time.combine(vandaag, i.begintijd)).total_seconds()
        step = timedelta(minutes=1)
        
        #controleer wie is ingelogd
        if str(Personen.objects.filter(account_id=response.user.id)[0]) == str(i.persoon):   

            for j in range(0, int(seconds)+60, int(step.total_seconds())):
                persoon_ingelogd_minuten_delta_list.append([ (i.persoon.naam), start + (timedelta(seconds=j))])
                persoon_ingelogd_dict[i.persoon.naam] = persoon_ingelogd_minuten_delta_list

        else:

            for k in range(0, int(seconds)+60, int(step.total_seconds())):
                persoon_overige_minuten_delta_list.append([ (i.persoon.naam), start + (timedelta(seconds=k))])
                persoon_overige_dict[i.persoon.naam] = persoon_overige_minuten_delta_list
```

Output ``` persoon_ingelogd_dict```

```python
Coen [['Coen', datetime.datetime(2021, 3, 7, 9, 15)], ['Coen', datetime.datetime(2021, 3, 7, 9, 16)], ['Coen', datetime.datetime(2021, 3, 7, 9, 17)], ['Coen', datetime.datetime(2021, 3, 7, 9, 18)], ['Coen', datetime.datetime(2021, 3, 7, 9, 19)], ['Coen', datetime.datetime(2021, 3, 7, 9, 20)], ['Coen', datetime.datetime(2021, 3, 7, 9, 21)], ['Coen', datetime.datetime(2021, 3, 7, 9, 22)], ['Coen', datetime.datetime(2021, 3, 7, 9, 23)], ['Coen', datetime.datetime(2021, 3, 7, 9, 24)], ['Coen', datetime.datetime(2021, 3, 7, 9, 25)], ['Coen', datetime.datetime(2021, 3, 7, 9, 26)], ['Coen', datetime.datetime(2021, 3, 7, 9, 27)], ['Coen', datetime.datetime(2021, 3, 7, 9, 28)], ['Coen', datetime.datetime(2021, 3, 7, 9, 29)], ['Coen', datetime.datetime(2021, 3, 7, 9, 30)], ['Coen', datetime.datetime(2021, 3, 7, 9, 31)], ['Coen', datetime.datetime(2021, 3, 7, 9, 32)], ['Coen', datetime.datetime(2021, 3, 7, 9, 33)], ['Coen', datetime.datetime(2021, 3, 7, 9, 34)], ['Coen', datetime.datetime(2021, 3, 7, 9, 35)], ['Coen', datetime.datetime(2021, 3, 7, 9, 36)], ['Coen', datetime.datetime(2021, 3, 7, 9, 37)], ['Coen', datetime.datetime(2021, 3, 7, 9, 38)], ['Coen', datetime.datetime(2021, 3, 7, 9, 39)], ['Coen', datetime.datetime(2021, 3, 7, 9, 40)], ['Coen', datetime.datetime(2021, 3, 7, 9, 41)], ['Coen', datetime.datetime(2021, 3, 7, 9, 42)], ['Coen', datetime.datetime(2021, 3, 7, 9, 43)], ['Coen', datetime.datetime(2021, 3, 7, 9, 44)], ['Coen', datetime.datetime(2021, 3, 7, 9, 45)], ['Coen', datetime.datetime(2021, 3, 7, 9, 46)], ['Coen', datetime.datetime(2021, 3, 7, 9, 47)], ['Coen', datetime.datetime(2021, 3, 7, 9, 48)], ['Coen', datetime.datetime(2021, 3, 7, 9, 49)], ['Coen', datetime.datetime(2021, 3, 7, 9, 50)], ['Coen', datetime.datetime(2021, 3, 7, 9, 51)], ['Coen', datetime.datetime(2021, 3, 7, 9, 52)], ['Coen', datetime.datetime(2021, 3, 7, 9, 53)], ['Coen', datetime.datetime(2021, 3, 7, 9, 54)], ['Coen', datetime.datetime(2021, 3, 
7, 9, 55)], ['Coen', datetime.datetime(2021, 3, 7, 9, 56)], ['Coen', datetime.datetime(2021, 3, 7, 9, 57)], ['Coen', datetime.datetime(2021, 3, 7, 9, 58)], ['Coen', datetime.datetime(2021, 3, 7, 9, 59)], ['Coen', datetime.datetime(2021, 3, 7, 10, 0)], ['Coen', datetime.datetime(2021, 3, 7, 10, 1)], ['Coen', datetime.datetime(2021, 3, 7, 10, 2)], ['Coen', datetime.datetime(2021, 3, 7, 10, 3)], ['Coen', datetime.datetime(2021, 3, 7, 10, 4)], ['Coen', datetime.datetime(2021, 3, 7, 10, 5)], ['Coen', datetime.datetime(2021, 3, 7, 10, 6)], ['Coen', datetime.datetime(2021, 3, 7, 10, 7)], ['Coen', datetime.datetime(2021, 3, 7, 10, 8)], ['Coen', datetime.datetime(2021, 3, 7, 10, 9)], ['Coen', datetime.datetime(2021, 3, 7, 10, 10)], ['Coen', datetime.datetime(2021, 3, 7, 10, 11)], ['Coen', datetime.datetime(2021, 3, 7, 10, 12)], ['Coen', datetime.datetime(2021, 3, 7, 10, 13)], ['Coen', datetime.datetime(2021, 3, 7, 10, 14)], ['Coen', datetime.datetime(2021, 3, 7, 10, 15)], ['Coen', datetime.datetime(2021, 3, 7, 10, 16)], ['Coen', datetime.datetime(2021, 3, 7, 10, 17)], ['Coen', datetime.datetime(2021, 3, 7, 10, 18)], ['Coen', datetime.datetime(2021, 3, 7, 10, 19)], ['Coen', datetime.datetime(2021, 3, 7, 10, 20)], ['Coen', datetime.datetime(2021, 3, 7, 10, 21)], ['Coen', datetime.datetime(2021, 3, 
7, 10, 22)], ['Coen', datetime.datetime(2021, 3, 7, 10, 23)], ['Coen', datetime.datetime(2021, 3, 7, 10, 24)], ['Coen', datetime.datetime(2021, 3, 7, 10, 25)], ['Coen', datetime.datetime(2021, 3, 7, 10, 26)], ['Coen', 
datetime.datetime(2021, 3, 7, 10, 27)], ['Coen', datetime.datetime(2021, 3, 7, 10, 28)], ['Coen', datetime.datetime(2021, 3, 7, 10, 29)], ['Coen', datetime.datetime(2021, 3, 7, 10, 30)]]
```

De volgende stap is om te controleren in deze list waar de minuten overeenkomen per persoon:

Output ``` persoon_overige_dict```

```
Overige [['Harm', datetime.datetime(2021, 3, 7, 9, 0)], ['Harm', datetime.datetime(2021, 3, 7, 9, 1)], ['Harm', datetime.datetime(2021, 3, 7, 9, 2)], ['Harm', datetime.datetime(2021, 3, 7, 9, 3)], ['Harm', datetime.datetime(2021, 3, 7, 9, 4)], ['Harm', datetime.datetime(2021, 3, 7, 9, 5)], ['Harm', datetime.datetime(2021, 3, 7, 9, 6)], ['Harm', datetime.datetime(2021, 3, 7, 9, 7)], ['Harm', datetime.datetime(2021, 3, 7, 9, 8)], ['Harm', datetime.datetime(2021, 3, 7, 9, 9)], ['Harm', datetime.datetime(2021, 3, 7, 9, 10)], ['Harm', datetime.datetime(2021, 3, 7, 9, 11)], ['Harm', datetime.datetime(2021, 3, 7, 9, 12)], ['Harm', datetime.datetime(2021, 3, 7, 9, 13)], ['Harm', datetime.datetime(2021, 3, 7, 9, 14)], ['Harm', datetime.datetime(2021, 3, 7, 9, 15)], ['Harm', datetime.datetime(2021, 3, 7, 9, 16)], ['Harm', datetime.datetime(2021, 3, 7, 9, 17)], ['Harm', 
datetime.datetime(2021, 3, 7, 9, 18)], ['Harm', datetime.datetime(2021, 3, 7, 9, 19)], ['Harm', datetime.datetime(2021, 3, 7, 9, 20)], ['Harm', datetime.datetime(2021, 3, 7, 9, 21)], ['Harm', datetime.datetime(2021, 3, 7, 9, 22)], ['Harm', datetime.datetime(2021, 3, 7, 9, 23)], ['Harm', datetime.datetime(2021, 3, 7, 9, 24)], ['Harm', datetime.datetime(2021, 3, 7, 9, 25)], ['Harm', datetime.datetime(2021, 3, 7, 9, 26)], ['Harm', datetime.datetime(2021, 3, 7, 9, 27)], ['Harm', datetime.datetime(2021, 3, 7, 9, 28)], ['Harm', datetime.datetime(2021, 3, 7, 9, 29)], ['Harm', datetime.datetime(2021, 3, 7, 9, 30)], ['Harm', datetime.datetime(2021, 3, 7, 
9, 31)], ['Harm', datetime.datetime(2021, 3, 7, 9, 32)], ['Harm', datetime.datetime(2021, 3, 7, 9, 33)], ['Harm', datetime.datetime(2021, 3, 7, 9, 34)], ['Harm', datetime.datetime(2021, 3, 7, 9, 35)], ['Harm', datetime.datetime(2021, 3, 7, 9, 36)], ['Harm', datetime.datetime(2021, 3, 7, 9, 37)], ['Harm', datetime.datetime(2021, 3, 7, 9, 38)], ['Harm', datetime.datetime(2021, 3, 7, 9, 39)], ['Harm', datetime.datetime(2021, 3, 7, 9, 40)], ['Harm', datetime.datetime(2021, 3, 7, 9, 41)], ['Harm', datetime.datetime(2021, 3, 7, 9, 42)], ['Harm', datetime.datetime(2021, 3, 7, 9, 43)], ['Harm', datetime.datetime(2021, 3, 7, 9, 44)], ['Harm', datetime.datetime(2021, 3, 7, 9, 45)], ['Harm', datetime.datetime(2021, 3, 7, 9, 46)], ['Harm', datetime.datetime(2021, 3, 7, 9, 47)], ['Harm', datetime.datetime(2021, 3, 7, 9, 48)], ['Harm', datetime.datetime(2021, 3, 7, 9, 49)], ['Harm', datetime.datetime(2021, 3, 7, 9, 50)], ['Harm', datetime.datetime(2021, 3, 7, 9, 51)], ['Harm', datetime.datetime(2021, 3, 7, 9, 52)], ['Harm', datetime.datetime(2021, 3, 7, 9, 53)], ['Harm', datetime.datetime(2021, 3, 7, 9, 54)], ['Harm', datetime.datetime(2021, 3, 7, 9, 55)], ['Harm', datetime.datetime(2021, 3, 7, 9, 56)], ['Harm', datetime.datetime(2021, 3, 7, 9, 57)], ['Harm', datetime.datetime(2021, 3, 7, 9, 58)], ['Harm', datetime.datetime(2021, 3, 7, 9, 59)], ['Harm', datetime.datetime(2021, 3, 7, 10, 0)], ['Thijs', datetime.datetime(2021, 3, 7, 9, 30)], ['Thijs', datetime.datetime(2021, 3, 7, 9, 31)], ['Thijs', datetime.datetime(2021, 3, 7, 9, 32)], ['Thijs', datetime.datetime(2021, 3, 7, 9, 33)], ['Thijs', datetime.datetime(2021, 3, 7, 9, 34)], ['Thijs', datetime.datetime(2021, 3, 7, 9, 35)], ['Thijs', datetime.datetime(2021, 3, 7, 9, 36)], 
['Thijs', datetime.datetime(2021, 3, 7, 9, 37)], ['Thijs', datetime.datetime(2021, 3, 7, 9, 38)], ['Thijs', datetime.datetime(2021, 3, 7, 9, 39)], ['Thijs', datetime.datetime(2021, 3, 7, 9, 40)], ['Thijs', datetime.datetime(2021, 3, 7, 9, 41)], ['Thijs', datetime.datetime(2021, 3, 7, 9, 42)], ['Thijs', datetime.datetime(2021, 3, 7, 9, 43)], ['Thijs', datetime.datetime(2021, 3, 7, 9, 44)], ['Thijs', datetime.datetime(2021, 3, 7, 9, 45)], ['Thijs', datetime.datetime(2021, 3, 7, 9, 46)], ['Thijs', datetime.datetime(2021, 3, 7, 9, 47)], ['Thijs', datetime.datetime(2021, 3, 7, 9, 48)], ['Thijs', datetime.datetime(2021, 3, 7, 9, 49)], ['Thijs', datetime.datetime(2021, 3, 7, 9, 50)], ['Thijs', datetime.datetime(2021, 3, 7, 9, 51)], ['Thijs', datetime.datetime(2021, 3, 7, 9, 52)], ['Thijs', datetime.datetime(2021, 3, 7, 9, 53)], ['Thijs', datetime.datetime(2021, 3, 7, 9, 54)], ['Thijs', datetime.datetime(2021, 3, 7, 9, 55)], ['Thijs', datetime.datetime(2021, 3, 7, 9, 56)], ['Thijs', datetime.datetime(2021, 3, 7, 9, 57)], ['Thijs', datetime.datetime(2021, 3, 7, 9, 58)], ['Thijs', datetime.datetime(2021, 3, 7, 9, 59)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 0)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 1)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 2)], ['Thijs', datetime.datetime(2021, 
3, 7, 10, 3)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 4)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 5)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 6)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 7)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 8)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 9)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 10)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 11)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 12)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 13)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 14)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 15)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 16)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 17)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 18)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 19)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 20)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 21)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 22)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 23)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 24)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 25)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 26)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 27)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 28)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 29)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 30)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 31)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 32)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 33)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 34)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 35)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 36)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 37)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 38)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 39)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 40)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 41)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 42)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 43)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 44)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 45)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 46)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 47)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 48)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 49)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 50)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 51)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 52)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 53)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 54)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 55)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 56)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 57)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 58)], ['Thijs', datetime.datetime(2021, 3, 7, 10, 59)], ['Thijs', datetime.datetime(2021, 3, 7, 11, 0)]]

```
