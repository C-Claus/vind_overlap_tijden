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

Er zijn meerdere manieren mogelijk om de raakvlakken te vinden, mijn aanpak is door eerst per persoon de tijddeltas te definieren als een lijst van minuten. Deze lijst stop ik ik een dictionary.  waarbij: key=persoon, value=tijddelta_minuten


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
    
    persoon_naam_overige_minuten_delta_list = []

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
                #voor later gebruik, aan het eind van he script
                persoon_naam_overige_minuten_delta_list.append([i.persoon, start + (timedelta(seconds=k))])
                
                persoon_overige_minuten_delta_list.append(start + (timedelta(seconds=k)))
               
                #door 'Overige' als key te gebruiken wordt er maar één dictionary gemaakt. 
                persoon_overige_dict['Overige'] = persoon_overige_minuten_delta_list
```

Output ``` persoon_ingelogd_dict```

```python
Coen [datetime.datetime(2021, 3, 7, 9, 15), datetime.datetime(2021, 3, 7, 9, 16), datetime.datetime(2021, 3, 7, 9, 17), datetime.datetime(2021, 3, 7, 9, 18), datetime.datetime(2021, 3, 7, 9, 19), datetime.datetime(2021, 3, 7, 9, 20), datetime.datetime(2021, 3, 7, 9, 21), datetime.datetime(2021, 3, 7, 9, 22), datetime.datetime(2021, 3, 7, 9, 23), datetime.datetime(2021, 3, 7, 9, 24), datetime.datetime(2021, 3, 7, 9, 25), datetime.datetime(2021, 3, 7, 9, 26), datetime.datetime(2021, 3, 7, 9, 27), datetime.datetime(2021, 3, 7, 9, 28), datetime.datetime(2021, 3, 7, 9, 29), datetime.datetime(2021, 3, 7, 9, 30), datetime.datetime(2021, 3, 7, 9, 31), datetime.datetime(2021, 3, 7, 9, 32), datetime.datetime(2021, 3, 7, 9, 33), datetime.datetime(2021, 3, 7, 9, 34), datetime.datetime(2021, 3, 7, 9, 35), datetime.datetime(2021, 3, 7, 9, 36), datetime.datetime(2021, 3, 7, 
9, 37), datetime.datetime(2021, 3, 7, 9, 38), datetime.datetime(2021, 3, 7, 9, 39), datetime.datetime(2021, 3, 7, 9, 40), datetime.datetime(2021, 3, 7, 9, 41), datetime.datetime(2021, 3, 7, 9, 42), datetime.datetime(2021, 3, 7, 9, 43), datetime.datetime(2021, 3, 7, 9, 44), datetime.datetime(2021, 3, 7, 9, 45), datetime.datetime(2021, 3, 7, 9, 46), datetime.datetime(2021, 3, 7, 9, 47), datetime.datetime(2021, 3, 7, 9, 48), datetime.datetime(2021, 3, 7, 9, 49), datetime.datetime(2021, 3, 7, 9, 50), datetime.datetime(2021, 3, 7, 9, 51), datetime.datetime(2021, 3, 7, 9, 52), datetime.datetime(2021, 3, 7, 9, 53), datetime.datetime(2021, 3, 7, 9, 54), 
datetime.datetime(2021, 3, 7, 9, 55), datetime.datetime(2021, 3, 7, 9, 56), datetime.datetime(2021, 3, 7, 9, 57), datetime.datetime(2021, 3, 7, 9, 58), datetime.datetime(2021, 3, 7, 9, 59), datetime.datetime(2021, 3, 7, 10, 0), datetime.datetime(2021, 3, 7, 10, 1), datetime.datetime(2021, 3, 7, 10, 2), datetime.datetime(2021, 3, 7, 10, 3), datetime.datetime(2021, 3, 7, 10, 4), datetime.datetime(2021, 3, 7, 10, 5), datetime.datetime(2021, 3, 7, 10, 6), datetime.datetime(2021, 3, 7, 10, 7), datetime.datetime(2021, 3, 7, 10, 8), datetime.datetime(2021, 3, 7, 10, 9), datetime.datetime(2021, 3, 7, 10, 10), datetime.datetime(2021, 3, 7, 10, 11), datetime.datetime(2021, 3, 7, 10, 12), datetime.datetime(2021, 3, 7, 10, 13), datetime.datetime(2021, 3, 7, 10, 14), datetime.datetime(2021, 3, 7, 10, 15), datetime.datetime(2021, 3, 7, 10, 16), datetime.datetime(2021, 3, 7, 10, 17), datetime.datetime(2021, 3, 7, 10, 18), datetime.datetime(2021, 3, 7, 10, 19), datetime.datetime(2021, 3, 7, 10, 20), datetime.datetime(2021, 3, 7, 10, 21), datetime.datetime(2021, 3, 7, 10, 22), datetime.datetime(2021, 3, 7, 10, 23), datetime.datetime(2021, 3, 7, 10, 24), datetime.datetime(2021, 3, 7, 10, 25), datetime.datetime(2021, 3, 7, 10, 26), datetime.datetime(2021, 3, 7, 10, 27), datetime.datetime(2021, 3, 7, 10, 28), datetime.datetime(2021, 3, 7, 10, 29), datetime.datetime(2021, 3, 7, 10, 30)]
```

De volgende stap is om te controleren in deze list waar de minuten overeenkomen per persoon:

Output ``` persoon_overige_dict```

```
Overige [datetime.datetime(2021, 3, 7, 9, 0), datetime.datetime(2021, 3, 7, 9, 1), datetime.datetime(2021, 3, 7, 9, 2), datetime.datetime(2021, 3, 7, 9, 3), datetime.datetime(2021, 3, 7, 9, 4), datetime.datetime(2021, 
3, 7, 9, 5), datetime.datetime(2021, 3, 7, 9, 6), datetime.datetime(2021, 3, 7, 9, 7), datetime.datetime(2021, 3, 7, 9, 8), datetime.datetime(2021, 3, 7, 9, 9), datetime.datetime(2021, 3, 7, 9, 10), datetime.datetime(2021, 3, 7, 9, 11), datetime.datetime(2021, 3, 7, 9, 12), datetime.datetime(2021, 3, 7, 9, 13), datetime.datetime(2021, 3, 7, 9, 14), datetime.datetime(2021, 3, 7, 9, 15), datetime.datetime(2021, 3, 7, 9, 16), datetime.datetime(2021, 3, 7, 9, 17), datetime.datetime(2021, 3, 7, 9, 18), datetime.datetime(2021, 3, 7, 9, 19), datetime.datetime(2021, 3, 7, 9, 20), datetime.datetime(2021, 3, 7, 9, 21), datetime.datetime(2021, 3, 7, 9, 22), datetime.datetime(2021, 3, 7, 9, 23), datetime.datetime(2021, 3, 7, 9, 24), datetime.datetime(2021, 3, 7, 9, 25), datetime.datetime(2021, 3, 7, 9, 26), datetime.datetime(2021, 3, 7, 9, 27), datetime.datetime(2021, 3, 
7, 9, 28), datetime.datetime(2021, 3, 7, 9, 29), datetime.datetime(2021, 3, 7, 9, 30), datetime.datetime(2021, 3, 7, 9, 31), datetime.datetime(2021, 3, 7, 9, 32), datetime.datetime(2021, 3, 7, 9, 33), datetime.datetime(2021, 3, 7, 9, 34), datetime.datetime(2021, 3, 7, 9, 35), datetime.datetime(2021, 3, 7, 9, 36), datetime.datetime(2021, 3, 7, 9, 37), datetime.datetime(2021, 3, 7, 9, 38), datetime.datetime(2021, 3, 7, 9, 39), datetime.datetime(2021, 3, 7, 9, 40), datetime.datetime(2021, 3, 7, 9, 41), datetime.datetime(2021, 3, 7, 9, 42), datetime.datetime(2021, 3, 7, 9, 43), datetime.datetime(2021, 3, 7, 9, 44), datetime.datetime(2021, 3, 7, 9, 45), datetime.datetime(2021, 3, 7, 9, 46), datetime.datetime(2021, 3, 7, 9, 47), datetime.datetime(2021, 3, 7, 9, 48), datetime.datetime(2021, 3, 7, 9, 49), datetime.datetime(2021, 3, 7, 9, 50), datetime.datetime(2021, 3, 7, 9, 51), datetime.datetime(2021, 3, 7, 9, 52), datetime.datetime(2021, 3, 7, 9, 53), datetime.datetime(2021, 3, 7, 9, 54), datetime.datetime(2021, 3, 7, 9, 55), datetime.datetime(2021, 3, 7, 9, 56), datetime.datetime(2021, 3, 7, 9, 57), datetime.datetime(2021, 3, 7, 9, 58), datetime.datetime(2021, 3, 7, 9, 59), datetime.datetime(2021, 3, 7, 10, 0), datetime.datetime(2021, 3, 7, 9, 30), datetime.datetime(2021, 3, 7, 9, 31), datetime.datetime(2021, 3, 7, 9, 32), datetime.datetime(2021, 3, 7, 9, 33), datetime.datetime(2021, 3, 7, 9, 34), datetime.datetime(2021, 3, 7, 9, 35), datetime.datetime(2021, 3, 7, 9, 36), datetime.datetime(2021, 3, 7, 9, 
37), datetime.datetime(2021, 3, 7, 9, 38), datetime.datetime(2021, 3, 7, 9, 39), datetime.datetime(2021, 3, 7, 9, 40), datetime.datetime(2021, 3, 7, 9, 41), datetime.datetime(2021, 3, 7, 9, 42), datetime.datetime(2021, 3, 7, 9, 43), datetime.datetime(2021, 3, 7, 9, 44), datetime.datetime(2021, 3, 7, 9, 45), datetime.datetime(2021, 3, 7, 9, 46), datetime.datetime(2021, 3, 7, 9, 47), datetime.datetime(2021, 3, 7, 9, 48), datetime.datetime(2021, 3, 7, 9, 49), datetime.datetime(2021, 3, 7, 9, 50), datetime.datetime(2021, 3, 7, 9, 51), datetime.datetime(2021, 3, 7, 9, 52), datetime.datetime(2021, 3, 7, 9, 53), datetime.datetime(2021, 3, 7, 9, 54), datetime.datetime(2021, 3, 7, 9, 55), datetime.datetime(2021, 3, 7, 9, 56), datetime.datetime(2021, 3, 7, 9, 57), datetime.datetime(2021, 3, 7, 9, 58), datetime.datetime(2021, 3, 7, 9, 59), datetime.datetime(2021, 3, 7, 10, 0), datetime.datetime(2021, 3, 7, 10, 1), datetime.datetime(2021, 3, 7, 10, 2), datetime.datetime(2021, 3, 7, 10, 3), datetime.datetime(2021, 3, 7, 10, 4), datetime.datetime(2021, 3, 7, 10, 5), datetime.datetime(2021, 3, 7, 10, 6), datetime.datetime(2021, 3, 7, 10, 7), datetime.datetime(2021, 3, 7, 10, 8), datetime.datetime(2021, 3, 7, 10, 9), datetime.datetime(2021, 3, 7, 10, 10), datetime.datetime(2021, 3, 7, 10, 11), datetime.datetime(2021, 3, 7, 10, 12), datetime.datetime(2021, 3, 7, 10, 13), datetime.datetime(2021, 3, 7, 10, 14), datetime.datetime(2021, 3, 7, 10, 15), datetime.datetime(2021, 3, 7, 10, 16), datetime.datetime(2021, 3, 7, 10, 17), datetime.datetime(2021, 3, 7, 10, 18), datetime.datetime(2021, 3, 7, 10, 19), datetime.datetime(2021, 3, 7, 10, 20), datetime.datetime(2021, 3, 7, 10, 21), datetime.datetime(2021, 3, 7, 10, 22), datetime.datetime(2021, 3, 7, 10, 23), datetime.datetime(2021, 3, 7, 10, 24), datetime.datetime(2021, 3, 7, 10, 25), datetime.datetime(2021, 3, 7, 10, 26), datetime.datetime(2021, 3, 7, 10, 27), datetime.datetime(2021, 3, 7, 10, 28), 
datetime.datetime(2021, 3, 7, 10, 29), datetime.datetime(2021, 3, 7, 10, 30), datetime.datetime(2021, 3, 7, 10, 31), datetime.datetime(2021, 3, 7, 10, 32), datetime.datetime(2021, 3, 7, 10, 33), datetime.datetime(2021, 3, 7, 10, 34), datetime.datetime(2021, 3, 7, 10, 35), datetime.datetime(2021, 3, 7, 10, 36), datetime.datetime(2021, 3, 7, 10, 37), datetime.datetime(2021, 3, 7, 10, 38), datetime.datetime(2021, 3, 7, 10, 39), datetime.datetime(2021, 3, 7, 10, 40), datetime.datetime(2021, 3, 7, 10, 41), datetime.datetime(2021, 3, 7, 10, 42), datetime.datetime(2021, 3, 7, 10, 43), datetime.datetime(2021, 3, 7, 10, 44), datetime.datetime(2021, 3, 7, 
10, 45), datetime.datetime(2021, 3, 7, 10, 46), datetime.datetime(2021, 3, 7, 10, 47), datetime.datetime(2021, 3, 7, 10, 48), datetime.datetime(2021, 3, 7, 10, 49), datetime.datetime(2021, 3, 7, 10, 50), datetime.datetime(2021, 3, 7, 10, 51), datetime.datetime(2021, 3, 7, 10, 52), datetime.datetime(2021, 3, 7, 10, 53), datetime.datetime(2021, 3, 7, 10, 54), datetime.datetime(2021, 3, 7, 10, 55), datetime.datetime(2021, 3, 7, 10, 56), datetime.datetime(2021, 3, 7, 10, 57), datetime.datetime(2021, 3, 7, 10, 58), datetime.datetime(2021, 3, 7, 10, 59), datetime.datetime(2021, 3, 7, 11, 0)]
```

Om de overlap te vinden kan de de volgende functie  gebruikt worden
```set``` geeft een list terug met alleen unieke waarden, door de twee listen te verenigen met &

```python
overlap_list = sorted(set(persoon_ingelogd_minuten_delta_list) & set(persoon_overige_minuten_delta_list))
```
Output van ```overlap_list```

```
2021-03-07 09:15:00
2021-03-07 09:16:00
2021-03-07 09:17:00
2021-03-07 09:18:00
2021-03-07 09:19:00
2021-03-07 09:20:00
2021-03-07 09:21:00
2021-03-07 09:22:00
2021-03-07 09:23:00
2021-03-07 09:24:00
2021-03-07 09:25:00
2021-03-07 09:26:00
2021-03-07 09:27:00
2021-03-07 09:28:00
2021-03-07 09:29:00
2021-03-07 09:30:00
2021-03-07 09:31:00
2021-03-07 09:32:00
2021-03-07 09:33:00
2021-03-07 09:34:00
2021-03-07 09:35:00
2021-03-07 09:36:00
2021-03-07 09:37:00
2021-03-07 09:38:00
2021-03-07 09:39:00
2021-03-07 09:40:00
2021-03-07 09:41:00
2021-03-07 09:42:00
2021-03-07 09:43:00
2021-03-07 09:44:00
2021-03-07 09:45:00
2021-03-07 09:46:00
2021-03-07 09:47:00
2021-03-07 09:48:00
2021-03-07 09:49:00
2021-03-07 09:50:00
2021-03-07 09:51:00
2021-03-07 09:52:00
2021-03-07 09:53:00
2021-03-07 09:54:00
2021-03-07 09:55:00
2021-03-07 09:56:00
2021-03-07 09:57:00
2021-03-07 09:58:00
2021-03-07 09:59:00
2021-03-07 10:00:00
2021-03-07 10:01:00
2021-03-07 10:02:00
2021-03-07 10:03:00
2021-03-07 10:04:00
2021-03-07 10:05:00
2021-03-07 10:06:00
2021-03-07 10:07:00
2021-03-07 10:08:00
2021-03-07 10:09:00
2021-03-07 10:10:00
2021-03-07 10:11:00
2021-03-07 10:12:00
2021-03-07 10:13:00
2021-03-07 10:14:00
2021-03-07 10:15:00
2021-03-07 10:16:00
2021-03-07 10:17:00
2021-03-07 10:18:00
2021-03-07 10:19:00
2021-03-07 10:20:00
2021-03-07 10:21:00
2021-03-07 10:22:00
2021-03-07 10:23:00
2021-03-07 10:24:00
2021-03-07 10:25:00
2021-03-07 10:26:00
2021-03-07 10:27:00
2021-03-07 10:28:00
2021-03-07 10:29:00
2021-03-07 10:30:00
```

Een visuele controle laat zien dat dit alle overlappende minuten van de gebruiker Coen zijn, het laat echter nog niet zien met wie de overlapping heeft plaatsgevonden.
Daarvoor maken we een aparte list waar we de persoon kunnen toevoegen

```python
overlap_per_naam_list = []

for j in overlap_list:
    for i in persoon_naam_overige_minuten_delta_list:
        if i[1] == j:
            overlap_per_naam_list.append( [i, [j]])

```


Output ```overlap_per_naam_list```

```python
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 15)]]
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 16)]]
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 17)]]
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 18)]]
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 19)]]
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 20)]]
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 21)]]
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 22)]]
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 23)]]
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 24)]]
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 25)]]
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 26)]]
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 27)]]
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 28)]]
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 29)]]
[<Personen: Thijs>, [datetime.datetime(2021, 3, 7, 9, 30)]]
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 30)]]
[<Personen: Thijs>, [datetime.datetime(2021, 3, 7, 9, 31)]]
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 31)]]
[<Personen: Thijs>, [datetime.datetime(2021, 3, 7, 9, 32)]]
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 32)]]
[<Personen: Thijs>, [datetime.datetime(2021, 3, 7, 9, 33)]]
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 33)]]
[<Personen: Thijs>, [datetime.datetime(2021, 3, 7, 9, 34)]]
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 34)]]
[<Personen: Thijs>, [datetime.datetime(2021, 3, 7, 9, 35)]]
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 35)]]
[<Personen: Thijs>, [datetime.datetime(2021, 3, 7, 9, 36)]]
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 36)]]
[<Personen: Thijs>, [datetime.datetime(2021, 3, 7, 9, 37)]]
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 37)]]
[<Personen: Thijs>, [datetime.datetime(2021, 3, 7, 9, 38)]]
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 38)]]
[<Personen: Thijs>, [datetime.datetime(2021, 3, 7, 9, 39)]]
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 39)]]
[<Personen: Thijs>, [datetime.datetime(2021, 3, 7, 9, 40)]]
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 40)]]
[<Personen: Thijs>, [datetime.datetime(2021, 3, 7, 9, 41)]]
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 41)]]
[<Personen: Thijs>, [datetime.datetime(2021, 3, 7, 9, 42)]]
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 42)]]
[<Personen: Thijs>, [datetime.datetime(2021, 3, 7, 9, 43)]]
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 43)]]
[<Personen: Thijs>, [datetime.datetime(2021, 3, 7, 9, 44)]]
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 44)]]
[<Personen: Thijs>, [datetime.datetime(2021, 3, 7, 9, 45)]]
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 45)]]
[<Personen: Thijs>, [datetime.datetime(2021, 3, 7, 9, 46)]]
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 46)]]
[<Personen: Thijs>, [datetime.datetime(2021, 3, 7, 9, 47)]]
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 47)]]
[<Personen: Thijs>, [datetime.datetime(2021, 3, 7, 9, 48)]]
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 48)]]
[<Personen: Thijs>, [datetime.datetime(2021, 3, 7, 9, 49)]]
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 49)]]
[<Personen: Thijs>, [datetime.datetime(2021, 3, 7, 9, 50)]]
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 50)]]
[<Personen: Thijs>, [datetime.datetime(2021, 3, 7, 9, 51)]]
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 51)]]
[<Personen: Thijs>, [datetime.datetime(2021, 3, 7, 9, 52)]]
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 52)]]
[<Personen: Thijs>, [datetime.datetime(2021, 3, 7, 9, 53)]]
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 53)]]
[<Personen: Thijs>, [datetime.datetime(2021, 3, 7, 9, 54)]]
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 54)]]
[<Personen: Thijs>, [datetime.datetime(2021, 3, 7, 9, 55)]]
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 55)]]
[<Personen: Thijs>, [datetime.datetime(2021, 3, 7, 9, 56)]]
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 56)]]
[<Personen: Thijs>, [datetime.datetime(2021, 3, 7, 9, 57)]]
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 57)]]
[<Personen: Thijs>, [datetime.datetime(2021, 3, 7, 9, 58)]]
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 58)]]
[<Personen: Thijs>, [datetime.datetime(2021, 3, 7, 9, 59)]]
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 59)]]
[<Personen: Thijs>, [datetime.datetime(2021, 3, 7, 10, 0)]]
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 10, 0)]]
```

Deze output geeft het antwoord waar de overlap zit, de output omzetten naar een voor mensen leesbare structuur gebeurt als volgt:

```python
#zet de list van lists om naar een dictionary met defaultdict, defaultdict kan een dynamische key bevatten ipv dict
overlap_dict = defaultdict(list)

for naam, minuten in overlap_per_naam_list:
    overlap_dict[naam].append(minuten)

overlap_persoon_minuut_list = overlap_dict.items()   


#fatsoeneer de lijst om mooi te renderen in HTML
render_persoon_minuut_list = []
for i in overlap_persoon_minuut_list:
    render_persoon_minuut_list.append([i[0], i[1][0], i[1][-1]])  
```

Output ```render_persoon_minuut_list``` Let wel op, hier zitten de intervallen niet meer tussen, dit zijn alleen de begin en eindtijden van de overlapping gemaakt ten behoeve voor leesbaarheid gebruiker. 

```
[<Personen: Coen>, [datetime.datetime(2021, 3, 7, 9, 15)], [datetime.datetime(2021, 3, 7, 10, 0)]]
[<Personen: Thijs>, [datetime.datetime(2021, 3, 7, 9, 30)], [datetime.datetime(2021, 3, 7, 10, 0)]]
```


# Eindproduct

<img src="https://github.com/C-Claus/vind_overlap_tijden/blob/main/eindresultaat.PNG" alt="drawing" width="1000"/>


# Conclusie

*  begintijd en eindtijd is bekend per persoon
*  hiermee kan een tijddelta gemaakt worden (een list van minuten ( eindtijd-begintijd))
*  deze list kan in een dictionary worden gestopt, waarbij de key de persoon is.
*  door alle minuten van de overige personen (de niet ingelogd persoon) in een lijst te stoppen, kunnen de items van de list vergeleken worden met de items van de list van het ingelogde persoon. de matchende list items is waar de persoon overlap heeft
*  door de minuten te vergelijken van de persoon tijddelta dictionary kan weer worden herleid met wie de ingelogde persoon tegelijkertijd afwezig is
*  mochten er intervallen zijn op een afwezigheidsregistratie dan ziet dit script dit ook omdat er per minuut wordt vergeleken
*  script werkt alleen op dagregistratie


