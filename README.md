# CST8917 Lab 03
## Elizabeth Kaganovsky (040956095)

### Demo link: https://youtu.be/hdAa1CzfwuY 

### Setup: Cloud Resources
Create the following resources in Azure:

**Resource Group**
- Name: `rg-serverless-lab`
- Region: Canada Central

**Service Bus Namespace**
- Name: `kaga0008-sb-namespace` (Must be unique)
- Pricing: Standard
After creation, go to Shared access policies > RootManageSharedAccessKey and copy the Primary Connection String and Primary Key.

**Booking Queue**
Create a queue in the service bus namespace.
- Name: `booking-queue`
- All else: Defaults

**Booking Results Topic**
Create a booking results topic in the service bus namespace.
- Name: `booking-results`
- All else: Defaults

**Filtered Subscriptions**
Create two filtered subscriptions.
confirmed-sub:
- Name: `ConfirmedFilter`
- Type: SQL Filter
- Expression: `sys.label = 'confirmed'`

rejected-sub
- Name: `RejectedFilter`
- Type: SQL Filter
- Expression: `sys.label = 'rejected'`

### Setup: Functions
- Create an Azure Functions Project using the provided `function_app.py`.
- Activate the environment from the provided `requirements.txt`.
- Rename `local.settings.example.json` to `local.sttings.json`.
- Deploy the Function App to Azure.

### Setup: Logic App
- Name: `process-booking`
- Plan: Consumption

Trigger: Service Bus Queue
- Name: `booking-sb-connection`
- Authentication: Connection string (paste from previous step)
- Queue name: `booking-queue`
- Check: Every 30 seconds

Next step: Decode and Parse the Queue Message
- Name: `Decode Message`
- Action: Compose
- Expression: `base64ToString(triggerBody()?['ContentData'])`

Next step: Parse the Booking JSON
- Name: `Parse Booking Request`
- Action: Data operations > Parse JSON
- Content: Dynamic content -> Outputs of Decode Message
- Schema: Use sample payload:
```
{
    "bookingId": "BK-0001",
    "customerName": "Jane Doe",
    "customerEmail": "jane@example.com",
    "vehicleType": "sedan",
    "pickupLocation": "Ottawa",
    "pickupDate": "2026-04-01",
    "returnDate": "2026-04-05",
    "notes": "GPS",
    "submittedAt": "2026-03-25T10:30:00.000Z"
}
```

Next Step: Call the Azure Function
- Name: `Call Check-Booking Function`
- Action: Azure Functions > Select created Function App
- Request Body: Dynamic content > Parse Booking Request > Body

Next Step: Parse the Function Response
- Name: `Parse Function Response`
- Action: Data operations > Parse JSON
- Schema: Use sample payload:
```
{
    "bookingId": "BK-0001",
    "customerName": "Jane Doe",
    "customerEmail": "jane@example.com",
    "status": "confirmed",
    "vehicleId": "V001",
    "vehicleType": "sedan",
    "location": "Ottawa",
    "pickupDate": "2026-04-01",
    "returnDate": "2026-04-05",
    "estimatedPrice": 200,
    "pricing": {
        "days": 4,
        "dailyRate": 45,
        "basePrice": 180,
        "addOns": ["GPS ($5/day)"],
        "addOnTotal": 20,
        "discount": 0,
        "estimatedPrice": 200
    },
    "reason": "Vehicle V001 (sedan) available in Ottawa"
}
```
- Update vehicleID to "type": ["string", "null"] and estimatedPrice to "type": ["integer", "null"]

Next Step: Add Conditional Branching
- Action: Condition
- Left: status from Parse Function Response
- Operator: is equal to
- Right: confirmed

True branch:
- To: `@{body('Parse_Function_Response')?['customerEmail']}`
- Subject: `FleetBook Confirmation -- @{body('Parse_Function_Response')?['bookingId']}`
- Body:
```
Your booking has been confirmed!

Booking ID: @{body('Parse_Function_Response')?['bookingId']}
Customer: @{body('Parse_Function_Response')?['customerName']}
Vehicle: @{body('Parse_Function_Response')?['vehicleId']} @{body('Parse_Function_Response')?['vehicleType']}
Location: @{body('Parse_Function_Response')?['location']}
Dates: @{body('Parse_Function_Response')?['pickupDate']} to @{body('Parse_Function_Response')?['returnDate']}
Estimated Total: @{body('Parse_Function_Response')?['estimatedPrice']}
Reason: @{body('Parse_Function_Response')?['reason']}

Thank you for choosing FleetBook!
```

Publish Confirmed to Topic
- Queue/topic name: `booking-results`
- Content: Call Check-Booking Function > Body
- Label: confirmed
- Content Type: application/json

False Branch:
- To: `@{body('Parse_Function_Response')?['customerEmail']}`
- Subject: `FleetBook -- Booking @{body('Parse_Function_Response')?['bookingId']} Could Not Be Confirmed`
- Body:
```
We're sorry, your booking could not be confirmed.

Booking ID: @{body('Parse_Function_Response')?['bookingId']}
Customer: @{body('Parse_Function_Response')?['customerName']}
Requested: @{body('Parse_Function_Response')?['vehicleType']} in @{body('Parse_Function_Response')?['location']}
Dates: @{body('Parse_Function_Response')?['pickupDate']} to @{body('Parse_Function_Response')?['returnDate']}
Reason: @{body('Parse_Function_Response')?['reason']}

Please try a different vehicle type or location.
```

Publish Rejected to Topic
- Queue/topic name: `booking-results`
- Content: Call Check-Booking Function > Body
- Label: rejected
- Content Type: application/json

### Setup: Web Client
- Open `client.html`
- Namespace Name: `kaga0008-sb-namespace`
- SAS Key: Primary Key from above step

Fill in the booking info and watch the emails roll in!

Tips:
-   If functions are not appearing in func. app:
    - Update requirements .txt
        - Do `pip freeze > requirements.text` to update 
    - Redeploy