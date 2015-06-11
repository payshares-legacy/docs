# Federation Overview

Payshares "federation" is a protocol which maps email style addressess to a payshares address. It's a way for Payshares software to resolve bob@yourdomain.com into gDSSa75HPagWcvQmwH7D51dT5DPmvsKL4q. Federated addresses provide an easy way for your users to share their Payshares address, using a syntax which interoperates across different domains supporting Payshares.

Additionally, federation can support destination tags. Just return the user's destination tag appended to the address in this format:
`<address>?dt=tag`. Example: bob@yourdomain.com -> gDSSa75HPagWcvQmwH7D51dT5DPmvsKL4q?dt=123.

# Supporting Federation

#### Step 1: Create payshares.txt file

Create a file called payshares.txt and put it at the root of your domain. In the payshares-lib javascript, the payshares.txt file is searched for in this order:

- `https://payshares.DOMAIN/payshares.txt`
- `https://DOMAIN/payshares.txt`
- `https://www.DOMAIN/payshares.txt`

#### Step 2: Add federation_url

Add the following to the payshares.txt file:

`[federation_url] https://api.yourdomain.com/federation`


#### Step 3: Implement federation url HTTP endpoint

The federation URL specified in your payshares.txt file should accept an HTTP GET request, with a query parameter “destination” to specify the username. It should return a JSON response body of this form:

federation_json: {
  destination: username,
  domain: domain,
  type: "federation_record",
  destination_address: address
}
