### 7/4/2018 - Sprint 195

#### TODO

* Schätzung verbleibende Zeit
  * Im moment Ball flach halten
  * Parallel arbeiten
  * Refactoring zum Schluss
* Merge master -> API
* PR für API

#### Notes

##### Misc

OrderPaymentState enum erweitern mit DEBTCOLLECTION und DEBTCOLLECTIONINSURED

=> OrderId: siehe confluence Test Cases

Gesamtpreis für deepsets ist falsch für die neu refactored API. D.h. wir  schicken dinge zu viel und bekommen ein grösseres total.

##### Merge master -> API -> PR

22 tests failed

VerifySimpleInjectorShopConfig lokal ausführen, bevor PR erstellt wird. Diese Fehler werden erst auf dem remote build erkannt.

OrderSearchService musste OrderRepository als Interface einbinden.

##### Work Split

| API                                                          | Dev    |
| ------------------------------------------------------------ | ------ |
| AccountingItem                                               | Zoltan |
| Zahlungsdetails                                              | Chrigi |
| Lieferstatus                                                 | Petri  |
| Zusatzinfo für Produktpositionen – Artikelzustand, Lieferant, Garantie |        |
| Actions pro Produkt – ob Button da ist + Link                |        |
| Actions zur Bestellung                                       |        |
| pdf?                                                         |        |

##### Zahlungsdetails API

![1530696340618](C:\Users\CHRIST~1.LAU\AppData\Local\Temp\1530696340618.png)

###### ApiModel

```json
{
    paymentDetails: {
        bills: [
            {
                id: 23456
                showLink: "/Invoice/ShowFromOrder/5803404",
            },
        	{...},
    		...
        ],
        invoices: [
            {
                id: 5803404,
                showLink: "/Invoice/ShowFromOrder/5803404", (?)
                date: "04-25-2018",
                amount: {
                    VatMoney
                },
                status: "PAID"
            },
        	{...},
    		...
        ],
        creditsAndRefunds: [
            {
                
            },
        	{...},
    		...      
        ]
    }
}
```

###### Notes

```c#
namespace Dg.AfterSales.Orders
{
    public class OrderService : IOrderService
    {
    ...
	public PaymentDetailsViewModel GetPaymentDetailViewModelOrNull(Id<Order> orderId,
    	                                                           int? userId,
        	                                                       int? customerId,
            	                                                   Language language)
	...
	}
}
```



### 7/5/2018 - Sprint 195

#### TODO

* Fix set recursion bug -> _done_
  * IsProductInPartnerservice check hat nach Refactoring gefehlt.
* Fix poweradapter no price bug -> _done_
  * Komischer check der überprüft ob ein item ein Poweradapter ist nd dann Preis 0 gibt entfernt.
* Login Logik prüfen.
  * Redirect bei SignOut -> _OK_
  * Reload orders on SignIn -> _OK_
  * Anmelden falls noch nicht angemeldet -> _OK_
  * Angemeldet bleiben -> _OK_
* Testing Bestellübersicht -> seems _OK_, needs testing before go live
* API Review durch Michael Malcharek -> _in Bearbeitung_
* Zahlungsdetails API

#### Notes

##### Misc

##### Login Logik

* Bug wenn die URL nicht lowecase ist, also z.B. /Order, /OrDeR
  * Gefixt von Nils -> _OK_
* CSS Feinschliff fehlt noch.
* sonst alles _OK_

##### Code Review

* IsRelease Check in ValidateUser verschoben.

##### Zahlungsdetails API

-> Siehe 7/4/2018 - Sprint 195

```csharp
// Api\OrderController.cs
/// <summary>
/// Returns the payment details of the customer for one specific order.
/// </summary>
/// <param name="userId">Identifier of the user</param>
/// <param name="orderId">Identifier of the order</param>
/// <returns>Paged list with all payment details for the order and customer</returns>
[Route("v1/users/{userId:int}/orders/{orderId:int}/paymentdetails")]
[HttpGet]
[SwaggerResponse(HttpStatusCode.OK, type: typeof(OrderResponse))]
[SwaggerResponse(HttpStatusCode.BadRequest, type: typeof(WebApiErrorResult))]
[SwaggerResponse(HttpStatusCode.Unauthorized, type: typeof(WebApiErrorResult))]
[ApiAuthenticate(requireLogin: true)]
public IHttpActionResult GetPaymentDetailsForOrder(
    [FromUri] int userId,
    [FromUri] int orderId)
{
    SetContext(Culture);
    var customerId = -1;
    if (!ValidateUser(userId: userId, loggedInUserId: webContext.UserInfo.UserId)){
        return Content(HttpStatusCode.Unauthorized, new WebApiErrorResult(modelStateDictionary: ModelState));
    }
    customerId = userToCustomerMappingRepository.GetCustomerId(userId: userId).ToInt();
    if (!ModelState.IsValid) {
        return Content(HttpStatusCode.BadRequest, new WebApiErrorResult(modelStateDictionary: ModelState));
    }
    return Ok(orderApiService.GetPaymentDetailsForCustomerAndOrder(customerId: customerId, userId: userId, orderId: orderId));
}
```

```csharp
// IOrderApiService.cs
PaymentDetailResponse GetPaymentDetailsForCustomerAndOrder(int customerId, int userId, int orderId);
// OrderApiService.cs
public PaymentDetailResponse GetPaymentDetailsForCustomerAndOrder(int customerId, int userId, int orderId)
{
    var language = DbCache.GetEntity(LanguageIds.English.ToTypedId<Language>());
    var commonParameters = commonParametersService.GetCommonParametersForCurrentContextAndDisplayParameters(
    	new AdditionalFilter(onlyActiveProducts: false),
    	null);
            
    var orderViewModel = orderService.GetOrderViewModelOrNull(orderId, userId, customerId, language, commonParameters);
    var invoices = new List<InvoiceApiModel>();
    foreach (var invoice in orderViewModel.PaymentDetailsViewModel.Invoices) {
        var invoiceApiModel = new InvoiceApiModel(
            invoice.Id,
            invoice.InvoiceDate,
            invoice.ApplyDiscount? invoice.AmountWithDiscount : invoice.Amount,
            invoice.Invoicing.PaymentState.GetName()); 
            invoices.Add(invoiceApiModel);
    }

    var paymentDetailsApiModel = new PaymentDetailsApiModel(null, invoices, null);
    var paymentDetailResponse = new PaymentDetailResponse(ReadOnlyList.Create(paymentDetailsApiModel));
    return paymentDetailResponse;
}
```

```csharp
// PaymentDetailResponse.cs
public class PaymentDetailResponse
{
    public IReadOnlyList<PaymentDetailsApiModel> PaymentDetails;

    public PaymentDetailResponse(IReadOnlyList<PaymentDetailsApiModel> paymentDetails)
    {
        PaymentDetails = paymentDetails;
    }
}
```

```csharp
// PaymentDetailsApiModel.cs
public class PaymentDetailsApiModel
{
    public IReadOnlyList<BillApiModel> Bills;
    public IReadOnlyList<InvoiceApiModel> Invoices;
    public IReadOnlyList<CreditAndRefundApiModel> CreditsAndRefunds;

    public PaymentDetailsApiModel(
        IReadOnlyList<BillApiModel> bills,
        IReadOnlyList<InvoiceApiModel> invoices,
        IReadOnlyList<CreditAndRefundApiModel> creditsAndRefunds)
    {
        Bills = bills;
        Invoices = invoices;
        CreditsAndRefunds = creditsAndRefunds;
    }
}
```

###### Current State (stashed)

```json
{
    "paymentDetails": [
        {
            "bills": null,
            "invoices": [
                {
                    "id": 6075543,
                    "date": "2018-06-22",
                    "amount": 138.2,
                    "name": "keine Bankzahlung"
                }
            ],
            "creditsAndRefunds": null
        }
    ]
}
```

### 7/6/2018 - Sprint 195

#### TODO

* PaymentDetail API
* MLE POTW
* MLE Code Review

#### Notes

##### PaymentDetail API

-> Siehe 7/5/2018 - Sprint 195

##### MLE POTW

* Alles wurde nach Machbarkeit und Value klassifiziert.
* "Lieferfrist im Shop verbessern" wurde als erstes Gildenthema ausgewählt
  * Bis zum nächsten POTW mal anschauen welche Daten wir in devinite haben.
  * Werden dann am 14/6/2018 zusammen in die DB schauen.

##### MLE Code Review

* Nicht PEP 8 konform, sollte aber kein Problem sein.
* Wird REST API so überhaupt noch gebraucht?
* basic_utils.py 3 mal kopiert, können wir wahrscheinlich im dg_ml_core einbauen

### 7/9/2018 - Sprint 195

#### TODO

* Abklärung Go-Live mit Isotopes
* PaymentDetail API

#### Notes

##### Go-Live

###### Overview

test: schon da
prod: https://jira.devinite.com/browseDO-242

###### Detail

test: https://jira.devinite.com/browse/DO-245
prod: https://jira.devinite.com/browse/DO-246

###### Notes

Requested meeting with Isotopes this afternoon

```
# Via akamai spoofing akamai IP
23.49.59.235 test-www.galaxus.ch
23.49.59.235 test-www.digitec.ch
```

1. GraphQL Deployen (sollte immer non breaking sein)
2. API (Devinite) Deployen (sollte immer non breaking sein für neue APIs)
3. Iso Deployen

##### PaymentDetails API

-> Siehe 7/6/2018 - Sprint 195

###### Current State

```json
{
    "paymentDetails": [
        {
            "bills": [
                {
                    "id": 13823726,
                    "date": "2018-05-17",
                    "amount": 0,
                    "amountOpen": 0,
                    "paymentMethod": "Buchung"
                },
                {
                    "id": 13823730,
                    "date": "2018-05-17",
                    "amount": 0,
                    "amountOpen": 0,
                    "paymentMethod": "Buchung"
                },
                {
                    "id": 13823739,
                    "date": "2018-05-18",
                    "amount": 54.16,
                    "amountOpen": 0,
                    "paymentMethod": "Maestro (national)"
                }
            ],
            "invoices": [
                {
                    "id": 5893644,
                    "date": "2018-05-16",
                    "amount": 88.64,
                    "state": "Ersetzt"
                },
                {
                    "id": 5893646,
                    "date": "2018-05-16",
                    "amount": 54.16,
                    "state": "bezahlt"
                }
            ],
            "creditsAndRefunds": []
        }
    ]
}
```

Now working on displaying information of [14403949](http://local-www.galaxus.ch/de/Order/PaymentDetails/14403949):

![1531124659285](C:\Users\CHRIST~1.LAU\AppData\Local\Temp\1531124659285.png)

This is Nils' order -> used all previous refunds to pay and had 34.48 left over which was given as a new refund/creditNote.

=> TODO handle credits and refunds.

###### Current State

```json
{
    "paymentDetails": [
        {
            "payments": [
                {
                    "id": 13805177,
                    "date": "2018-05-16",
                    "amount": 0,
                    "amountOpen": 0,
                    "paymentMethod": "Buchung"
                },
                {
                    "id": 13823730,
                    "date": "2018-05-17",
                    "amount": 0,
                    "amountOpen": 0,
                    "paymentMethod": "Buchung"
                }
            ],
            "invoices": [
                {
                    "id": 5792331,
                    "dueDate": "2018-04-24",
                    "invoiceDate": "2018-04-24",
                    "amount": -8.16,
                    "amountWithDiscount": -8.16
                },
                {
                    "id": 5858857,
                    "dueDate": "2018-05-08",
                    "invoiceDate": "2018-05-08",
                    "amount": -15.82,
                    "amountWithDiscount": -15.82
                },
                {
                    "id": 5858859,
                    "dueDate": "2018-05-08",
                    "invoiceDate": "2018-05-08",
                    "amount": -45.51,
                    "amountWithDiscount": -45.51
                },
                {
                    "id": 5858861,
                    "dueDate": "2018-05-08",
                    "invoiceDate": "2018-05-08",
                    "amount": -19.43,
                    "amountWithDiscount": -19.43
                },
                {
                    "id": 5858866,
                    "dueDate": "2018-05-08",
                    "invoiceDate": "2018-05-08",
                    "amount": -19.68,
                    "amountWithDiscount": -19.68
                },
                {
                    "id": 5886256,
                    "dueDate": "2018-05-15",
                    "invoiceDate": "2018-05-15",
                    "amount": -34.48,
                    "amountWithDiscount": -34.48
                },
                {
                    "id": 5893644,
                    "dueDate": "2018-06-05",
                    "invoiceDate": "2018-05-16",
                    "amount": 88.64,
                    "amountWithDiscount": 88.64
                }
            ],
            "creditsAndRefunds": [
                {
                    "id": 5886256,
                    "dueDate": "2018-05-15",
                    "invoiceDate": "2018-05-15",
                    "amount": -34.48,
                    "amountWithDiscount": -34.48,
                    "state": "Eingelöst in Auftrag 14463481"
                }
            ]
        }
    ]
}
```

Note: Renamed "bills" to payments.

###### Useful SQL queries

```mssql
select p.* from [order] o
join Customer c on c.id = o.Buyer_CustomerId
join Person p on p.CustomerId = c.Id
where o.id = 14403949

select o.* from [order] o
join Customer c on c.id = o.Buyer_CustomerId
join Person p on p.CustomerId = c.Id
where o.id = 14403949

select top 10 * from PaymentState ps
join view_DbContent cdbc on ps.Name_DbContentId = cdbc.Id
```

##### Refinement

###### [OSA-2869](https://jira.devinite.com/browse/OSA-2869)

* Soweit klar	
* 8 SP zu viel?
* /orders/[id]/additionalServices? oder einfach schon in /orders/[id]

###### [OSA-2891](https://jira.devinite.com/browse/OSA-2891)

* Petri arbeitet daran
* 5 SP left
* Sternchen fehlen noch in der Bearbeitung

###### [OSA-2882](https://jira.devinite.com/browse/OSA-2882)

* "meine" API
* invoices, payments und creditnotes
  * total berechnen im FE oder gleich von der API mitschicken...? Bin für mitschicken.
  * Enums machen oder einfach translated strings mitschicken?

###### [OSA-2870](https://jira.devinite.com/browse/OSA-2870)

* Müssen herausfinden welche logik bestimmt welche Actions möglich sind und diese kopieren.
  * Vielleicht bitmap?

##### [OSA-2863](https://jira.devinite.com/browse/OSA-2863)

* Müssen herausfinden welche logik bestimmt welche Actions möglich sind und diese kopieren.
  * Vielleicht bitmap?

### 7/10/2018 - Sprint 195

#### TODO

* Merge to master! (devinite, isomorph, graphql)0
* PaymentDetail API

#### Notes

##### Merge to master

* Some tests failed when building in VSTS => forgot to pull & push again.
* Waiting for GQL and ISO...

##### PaymentDetail API

-> see 7/9/2018 - Sprint 195

###### TestCases

https://conf.devinite.com/display/KUU/Testdaten

Muys Orders:

```mssql
select * from person p where p.CustomerId=696309
select * from [order] o where o.Buyer_CustomerId=696309
```

###### Notes

*payment ooptions*

![1531205758647](C:\Users\CHRIST~1.LAU\AppData\Local\Temp\1531205758647.png)

*payment state*

![1531206100663](C:\Users\CHRIST~1.LAU\AppData\Local\Temp\1531206100663.png)

### 7/11/2018 - Sprint 195

#### TODO

* PaymentDetail API
* Feedback Rudin
* ML Meeting
* Retro & Planning

#### Notes

##### PaymentDetail API

-> see 7/10/2018 - Sprint 195

Momentan Arbeit an den neuen Invisio Anpassungen
-> Schauen was von PaymentAPI kommt und was von DeliveryAPI.

Mit FE abklären ob die APIs so passen sollten.

##### Feedback Rudin

* Auf test-www kann man jetzt mit /order?userid=XXX bestellungen aller user in der übersicht ansehen,
* Die alte OrderOvervew kann man mit /Order/ordertestuser/XXX ansehen.

##### ML Meeting

* Python Code Doku & Refactoring _vor_ 1. August
* Git Workshop: Wann?

##### Retro & Planning

https://conf.devinite.com/pages/viewpage.action?pageId=126179522

Auto: kA, eins das schnell unterwegs ist aber nicht schön aussieht, schlecht wartbar ist und vielleicht explodieren könnte ;).

(+) Team zieht an einem Strang.
(+) Massnahmen wegen Zeitknappheit.

(-) PullRequests früher erstellen, ISOs früher abholen.
(-) Stakeholders mehr einbeziehen.

Zahlungsdetails Rest SP: Ich würde sagen BE Seite ~5. Braucht definitiv noch mehr TestCases. Sollte aber schon mal einigermassen funktionieren :)

### 7/12/2018 - Sprint 195

#### TODO

* PaymentDetail API
* MLE Gildentag abmachen
* Feedback Testsystem

#### Notes

##### PaymentDetail API

-> see 7/11/2018 - Sprint 195

###### Current State

```json
{
    "buyer": {
        "address": "NiXXXXXXX MiXXXXXXX, SiXXXXXXX, CH-80XXXXXXX LeXXXXXXX",
        "paymentOption": "0",
        "notes": ""
    },
    "price": {
        "amountTotal": { //TODO Split in total/paid
        	"amountIncl": 178.83,
        	"amountExcl": 166.05,
        	"currency": "CHF"
        },
        "amountPaid": {
            "amountIncl": 0.00,
        	"amountExcl": 0.00,
        	"currency": "CHF"
        }
    },
    "transactions": {
        "payments": [
            {
                "id": 13823726,
                "date": "2018-05-17",
                "amount": 0,
                "amountOpen": 0,
                "paymentMethod": "Buchung"
            },
            {
                "id": 13823730,
                "date": "2018-05-17",
                "amount": 0,
                "amountOpen": 0,
                "paymentMethod": "Buchung"
            },
            {
                "id": 13823739,
                "date": "2018-05-18",
                "amount": 54.16,
                "amountOpen": 0,
                "paymentMethod": "Maestro (national)"
            }
        ],
        "invoices": [
            {
                "id": 5893644,
                "dueDate": "2018-06-05",
                "invoiceDate": "2018-05-16",
                "amount": 88.64,
                "amountWithDiscount": 88.64
            },
            {
                "id": 5893646,
                "dueDate": "2018-06-05",
                "invoiceDate": "2018-05-16",
                "amount": 54.16,
                "amountWithDiscount": 54.16
            }
        ],
        "creditsAndRefunds": [
            {
                "id": 5858870,
                "dueDate": "2018-05-08",
                "invoiceDate": "2018-05-08",
                "amount": -90.19,
                "amountWithDiscount": -90.19,
                "state": "Gutschrift 5858870 (RMA-Auftrag 14382457)"
            },
            {
                "id": 5886256,
                "dueDate": "2018-05-15",
                "invoiceDate": "2018-05-15",
                "amount": -34.48,
                "amountWithDiscount": -34.48,
                "state": "Gutschrift 5886256 (Rest aus Gutschrift 5792331 (RMA-Auftrag 14198851))"
            }
        ]
    }
}
```

###### Current InVision

![1531393441260](C:\Users\CHRIST~1.LAU\AppData\Local\Temp\1531393441260.png)



##### MLE Gildentag

##### Feedback Testsystem

https://conf.devinite.com/display/KUU/06+Feedback+zum+Testsystem

* Kommentare geschrieben

##### Misc

* SignInAPI war kaputt -> fixed von den ISOs
* "Zurück zur Übersicht" Link auf Detailseite soll auf /Order zeigen -> Zoltan
* FightAmazon
  * Monica von Team Rocket hat sich bei uns gemeldet.
  * Haben bereits /Order auf DE getestet und wussten nicht, dass wir die Seiten iso machen.
  * Können nun nicht auf DE testen, bis ISOs die Mandanten-Story fertig haben...
  * TODO: Michaels informieren. 
    * Rudin ist informiert.

### 7/13/2018 - Sprint 195

#### TODO

* PaymentDetail API
* MLE POTW Meeting

#### Notes

##### PaymentDetail API

-> see 7/12/2018 - Sprint 195

###### Current State

```JSON
{
    "buyer": {
        "address": "NiXXXXXXX MiXXXXXXX, FrXXXXXXX, CH-80XXXXXXX ZüXXXXXXX",
        "paymentOption": "0",
        "notes": ""
    },
    "orderPrice": {
        "amountTotal": {
            "amountIncl": 74.12,
            "amountExcl": 68.82,
            "currency": "CHF"
        },
        "amountOpen": {
            "amount": "0.00",
            "currency": "CHF"
        }
    },
    "transactions": {
        "payments": [
            {
                "id": 13805177,
                "date": "2018-05-16",
                "amount": 0,
                "amountOpen": 0,
                "paymentMethod": "Buchung"
            },
            {
                "id": 13823730,
                "date": "2018-05-17",
                "amount": 0,
                "amountOpen": 0,
                "paymentMethod": "Buchung"
            }
        ],
        "invoices": [
            {
                "id": 5893644,
                "dueDate": "2018-06-05",
                "invoiceDate": "2018-05-16",
                "amount": 88.64,
                "amountWithDiscount": 88.64
            }
        ],
        "creditsAndRefunds": [
            {
                "id": 5792331,
                "dueDate": "2018-04-24",
                "invoiceDate": "2018-04-24",
                "amount": -8.16,
                "amountWithDiscount": -8.16,
                "state": "Gutschrift 5792331 (RMA-Auftrag 14198851)"
            },
            {
                "id": 5858857,
                "dueDate": "2018-05-08",
                "invoiceDate": "2018-05-08",
                "amount": -15.82,
                "amountWithDiscount": -15.82,
                "state": "Gutschrift 5858857 (RMA-Auftrag 14382378)"
            },
            {
                "id": 5858859,
                "dueDate": "2018-05-08",
                "invoiceDate": "2018-05-08",
                "amount": -45.51,
                "amountWithDiscount": -45.51,
                "state": "Gutschrift 5858859 (RMA-Auftrag 14382388)"
            },
            {
                "id": 5858861,
                "dueDate": "2018-05-08",
                "invoiceDate": "2018-05-08",
                "amount": -19.43,
                "amountWithDiscount": -19.43,
                "state": "Gutschrift 5858861 (RMA-Auftrag 14382402)"
            },
            {
                "id": 5858866,
                "dueDate": "2018-05-08",
                "invoiceDate": "2018-05-08",
                "amount": -19.68,
                "amountWithDiscount": -19.68,
                "state": "Gutschrift 5858866 (RMA-Auftrag 14382435)"
            },
            {
                "id": 5886256,
                "dueDate": "2018-05-15",
                "invoiceDate": "2018-05-15",
                "amount": -34.48,
                "amountWithDiscount": -34.48,
                "state": "Gutschrift 5886256 (Rest aus Gutschrift 5792331 (RMA-Auftrag 14198851))"
            }
        ]
    }
}
```

##### MLE POTW Meeting

Verschoben auf 7/17/2018

### 7/16/2018 - Sprint 195

#### TODO

* Order auf Test DE für @DavidUmbricht
* GoLive
* PaymentDetail API
* MLE Meeting Vorbereitrung

#### Notes

##### Order auf Test DE

Isotopes (@Daniel und @Roman arbeiten an Hotfix)

Momentan alle APIs auf DE kaput.

##### GoLive

###### Overview

test: schon da
prod: https://jira.devinite.com/browseDO-242

* Nachgefragt bei @HansDanielGraf wie lange das Live schalten dann dauern wird

###### Detail

test: https://jira.devinite.com/browse/DO-245
prod: https://jira.devinite.com/browse/DO-246

###### Zeitstrahl

Erstellt an der TeamWall von Goldfinger.

##### PaymentDetailAPI

* PR erstellt auf https://digitecgalaxus.visualstudio.com/_git/devinite/pullrequest/8360
  * Noch niocht ready für Review.

###### Current State

```json
{
    "buyer": {
        "address": "NiXXXXXXX MiXXXXXXX, FrXXXXXXX, CH-80XXXXXXX ZüXXXXXXX",
        "paymentOption": "BOOKING",
        "notes": null
    },
    "orderPrice": {
        "amountTotal": {
            "amountIncl": 74.12,
            "amountExcl": 68.82,
            "currency": "CHF"
        },
        "amountOpen": {
            "amountIncl": 0,
            "amountExcl": 0,
            "currency": "CHF"
        }
    },
    "transactions": {
        "payments": [
            {
                "id": 13805177,
                "date": "2018-05-16",
                "amount": 0,
                "amountOpen": 0,
                "paymentOption": "BOOKING",
                "paymentDirection": "INCOMING"
            },
            {
                "id": 13823730,
                "date": "2018-05-17",
                "amount": 0,
                "amountOpen": 0,
                "paymentOption": "BOOKING",
                "paymentDirection": "INCOMING"
            }
        ],
        "invoices": [
            {
                "id": 5893644,
                "dueDate": "2018-06-05",
                "invoiceDate": "2018-05-16",
                "amount": 88.64,
                "amountWithDiscount": 88.64
            }
        ],
        "creditsAndRefunds": [
            {
                "id": 5792331,
                "dueDate": "2018-04-24",
                "invoiceDate": "2018-04-24",
                "amount": -8.16,
                "amountWithDiscount": -8.16,
                "reference": "RMA-Auftrag 14198851"
            },
            {
                "id": 5858857,
                "dueDate": "2018-05-08",
                "invoiceDate": "2018-05-08",
                "amount": -15.82,
                "amountWithDiscount": -15.82,
                "reference": "RMA-Auftrag 14382378"
            },
            {
                "id": 5858859,
                "dueDate": "2018-05-08",
                "invoiceDate": "2018-05-08",
                "amount": -45.51,
                "amountWithDiscount": -45.51,
                "reference": "RMA-Auftrag 14382388"
            },
            {
                "id": 5858861,
                "dueDate": "2018-05-08",
                "invoiceDate": "2018-05-08",
                "amount": -19.43,
                "amountWithDiscount": -19.43,
                "reference": "RMA-Auftrag 14382402"
            },
            {
                "id": 5858866,
                "dueDate": "2018-05-08",
                "invoiceDate": "2018-05-08",
                "amount": -19.68,
                "amountWithDiscount": -19.68,
                "reference": "RMA-Auftrag 14382435"
            },
            {
                "id": 5886256,
                "dueDate": "2018-05-15",
                "invoiceDate": "2018-05-15",
                "amount": -34.48,
                "amountWithDiscount": -34.48,
                "reference": "Rest aus Gutschrift 5792331 (RMA-Auftrag 14198851)"
            }
        ]
    }
}
```

Bei creditsAndRefunds sollte die reference angeben, aus welcher order die Gutschrift kommt.
-> Scheint nur als string vorhanden zu sein...

##### MLE Meeting Vorbereitung

TBD

### 7/17/2018 - Sprint 195

#### TODO

* GoLive
* TestSystem DE
* PaymentDetail API
* MLE Meetings
  * Austausch mit AI Team (Daniel)
  * POTW -> Lieferdatum vorhersagen

#### Notes

##### GoLive

##### PaymentDetail API

##### MLE Meetings

###### Austausch AI Team

###### POTW

