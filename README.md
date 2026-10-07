# API & Webhook Test Suites 🎯

Automated Postman and Chai.js integration tests designed to validate webhook endpoints, ensure immediate HTTP handshakes, and enforce strict JSON response contracts for Make.com production pipelines.

## 🔍 Validation Focus
In production architecture, silent schema changes or latency spikes cause broken database rows and lost payloads. This suite enforces strict rules on the initial webhook handshake:
* **Latency SLA:** Fails any response taking longer than 1000ms to prevent upstream gateway timeouts.
* **HTTP Status:** Asserts immediate `200` or `201` resolutions before heavy asynchronous processing begins.
* **Strict Schema Enforcement:** Validates that the body parses as a pure JSON object (rejecting arrays and nulls) and checks exact string enum matches for `"status": "success"` and the designated payload message.

## 🛠 Core Test Assertions (Chai.js)
The collection runs the following embedded test scripts against the webhook URL on every payload drop:

```javascript
// Validate the successful response contract.
pm.test("Status code is 200 or 201", function () {
    pm.expect(pm.response.code).to.be.oneOf([200, 201]);
});

pm.test("Response time is under 1000 ms", function () {
    pm.expect(pm.response.responseTime).to.be.below(1000);
});

let responseJson;
pm.test("Body parses as JSON and is an object", function () {
    responseJson = pm.response.json();
    pm.expect(responseJson).to.be.an("object");
    pm.expect(responseJson).to.not.be.null;
    pm.expect(responseJson).to.not.be.an("array");
});

pm.test("status is required and exactly success", function () {
    responseJson = responseJson || pm.response.json();
    pm.expect(responseJson).to.have.property("status");
    pm.expect(responseJson.status).to.be.a("string");
    pm.expect(responseJson.status).to.eql("success");
});

pm.test("message is required and exactly Successfully caught payload", function () {
    responseJson = responseJson || pm.response.json();
    pm.expect(responseJson).to.have.property("message");
    pm.expect(responseJson.message).to.be.a("string");
    pm.expect(responseJson.message).to.eql("Successfully caught payload");
});

const expectedSchema = {
    type: "object",
    required: ["status", "message"],
    properties: {
        status: { type: "string", enum: ["success"] },
        message: { type: "string", enum: ["Successfully caught payload"] }
    }
};

pm.test("Response matches the required JSON schema", function () {
    pm.response.to.have.jsonSchema(expectedSchema);
});
