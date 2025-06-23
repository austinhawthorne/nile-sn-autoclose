# nile-sn-autoclose
Instructions and script for automatically opening and closing a ticket in ServiceNow based on the reception of a webhook alert from a Nile service

Nile provides a webhook that sends:

'''
Infrastructure alerts
Service alerts
Application alerts
Security alerts
'''

A number of these alerts are auto-remediated by the Nile service or otherwise.  Some customer's may want these alerts to open tickets automatically in ServiceNow and to close automatically if the service resolves the issue.  We can use the following fields within the Nile Webhook to accomplish this:

'''
'alertStatus' = Created or Resolved
'id' = unique alert identifier
'''

Follow these instructions to integrate Nile webhooks into ServiceNow with auto open/close function based on the 'alertStatus' field.

In ServiceNow:
1.  Under System Definitions > Tables > Incident > Columns click "New"
2.  Create a new dictionary entry with Type = String, Column label = alertId, Max length = 32
3.  Under System Web Services > Scripted Web Services > Scripted REST API click "New"
4.  Create a new API with name = nile_alerts, click "Submit"
5.  Find nile_alerts and click on it
6.  Under Resources click "New"
7.  Create a new resources with the Name = nile_alert_processing, copy the Resource Path and save for later, HTTP Method = POST, copy and paste the following into the script body and then click "Update":

'''
(function processRequest(request, response) {
    var body = request.body.data;

    var alertId = body["id"];
    var alertStatus = body["alertStatus"];

    if (!alertId || !alertStatus) {
        response.setStatus(400);
        response.setBody({ error: "Missing 'id' or 'alertStatus'" });
        return;
    }

    // Check for existing incident based on the unique alert ID
    var gr = new GlideRecord('incident');
    gr.addQuery('u_alert_id', alertId);  // Custom field in Incident table
    gr.query();

    if (alertStatus === "Created") {
        if (!gr.next()) {
            var newGr = new GlideRecord('incident');
            newGr.initialize();
            newGr.short_description = "Nile Alert - " + (body["alertType"] || "Unknown");
            newGr.description = 
                "Subject: " + body["alertSubject"] + "\n" +
                "Summary: " + body["alertSummary"] + "\n" +
                "Impact: " + body["impact"] + "\n" +
                "Customer: " + body["customer"] + "\n" +
                "Start Time: " + body["startTime"] + "\n" +
                "Duration: " + body["duration"] + "\n" +
                "Site/Building/Floor: " + body["site"] + " / " + body["building"] + " / " + body["floor"] + "\n" +
                "More Info: " + body["additionalInformation"];
            newGr.u_alert_id = alertId;
            newGr.urgency = 1;
            newGr.impact = 1;
            newGr.insert();
        }
    }

if (alertStatus === "Resolved") {
    var grResolve = new GlideRecord('incident');
    grResolve.addQuery('u_alert_id', alertId);
    grResolve.query();

    if (grResolve.next()) {
        grResolve.setValue("state", "7");
		grResolve.setValue("incident_state", "7" );
        grResolve.setValue("close_notes", "Resolved by Nile alert webhook.");
		grResolve.setValue("close_code", "5");
        grResolve.update();
        gs.info("Resolved incident with alert ID: " + alertId);
    } else {
        gs.info("No incident found to resolve for alert ID: " + alertId);
    }
}

    response.setStatus(200);
    response.setBody({ result: "Processed" });
})(request, response);
'''

8.  Convert your admin username and password to a Base64 encoded string to use for Basic Auth, for example:

'''
echo -n 'myuser:mypassword' | base64
'''

9.  In Nile, go to Global Settings > Integrations, select "Setup Integration" and select "Webhook"
10.  Setup new webhook with the following parameters, Name = "Service Now Alerts, Token = "Basic <your_base64_encoded_key", URL = http://<your_servicenow_instance_url><resource_path_from_step_7>, then click Next.
11.  Select all alerts you are interested in (select all except My Nile Incidents) then click Save.

To test, can use something like Curl or Postman to generate an alert with alertStatus = Created to create the ticket and then send again with alertStatus = Resolved.  You need to ensure the "id" is the same for both.  Here is a sample JSON body you can use for the test:

'''
{
  "version": "1.0",
  "id": "12345678-9abc-def-1234-56789abcdef1",
  "alertSubscriptionCategory": "Infrastructure Alerts",
  "alertType": "Internet",
  "alertStatus": "Created",
  "alertSubject": "Nile Alert Resolved [Internet]",
  "alertSummary": "Nile Service Block is not reachable from the Nile cloud. Please check the following: Power, Cable, Internet, and Firewall rules.",
  "impact": "Network connectivity may be impacted.",
  "customer": "Acme",
  "startTime": "2025-06-20T20:02:00+00:00",
  "duration": "2 days",
  "site": "Building 1",
  "building": "All",
  "floor": "All",
  "additionalInformation": "https://docs.nilesecure.com/customer-infrastructure-alerts"
}
'''
