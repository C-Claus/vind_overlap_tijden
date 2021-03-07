# Vind overlappende tijden
Een klein programma waarin medewerkers hun afwezigheden kunnen opgeven. Het programma toont vervolgens of afwezigheidsperiodes elkaar overlappen.

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


# urls.py

In de urls.py worden de views aangeroepen, hier kunnen de diverse urls ook worden naamgegeven

```python
urlpatterns =   [   path('aanwezigheid/<int:persoon>', views.aanwezigheid, name="aanwezigheid"),
                    path('aanwezigheids_overzicht/<int:persoon_id>/<int:jaar>/<int:maand>/<int:dag>', views.aanwezigheids_overzicht, name="aanwezigheids_overzicht"),
                    path('aanwezigheids_overzicht_per_dag', views.aanwezigheid_dagoverzicht, name="aanwezigheid_dagoverzicht"),
                 ]
```                    
