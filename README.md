Yes, you are correct. Typically, the file extension `.pem` is used for PEM format files. If your guide specifies the file should be in PEM format and the name should be `bundle.pem`, you should use that name for the concatenated file. Here's how you can do it:

1. **Convert the `.cer` file to `.pem` format**:
   Use OpenSSL to convert the `zScaler_Root_Certificate.cer` file to PEM format:

   ```bash
   openssl x509 -inform der -in zScaler_Root_Certificate.cer -out zScaler_Root_Certificate.pem
   ```

2. **Concatenate the two PEM files**:
   Concatenate the `design app-dev-ca.pem` and `zScaler_Root_Certificate.pem` files into a new file called `bundle.pem`:

   ```bash
   cat design\ app-dev-ca.pem zScaler_Root_Certificate.pem > bundle.pem
   ```

3. **Verify the contents of the `bundle.pem` file**:
   You can verify that the `bundle.pem` file contains both certificates by viewing its contents:

   ```bash
   cat bundle.pem
   ```

   The output should display the concatenated contents of both PEM files.

4. **Use the `bundle.pem` file**:
   You can now use the `bundle.pem` file as your Extra CA bundle file, which should contain both certificates in PEM format.

You can check if the `NODE_EXTRA_CA_CERTS` environment variable is correctly set and if Node.js is loading the `bundle.pem` file by running a simple Node.js script that prints out the loaded certificates. Here's a script you can use:

1. Create a new file named `check_ca_certs.js` with the following content:

   ```javascript
   const https = require('https');
   const fs = require('fs');

   // Read the bundle.pem file
   const ca = fs.readFileSync(process.env.NODE_EXTRA_CA_CERTS);

   // Set the global root certificates
   https.globalAgent.options.ca = ca;

   // Make a request to a website to trigger certificate verification
   https.get('https://www.google.com', (res) => {
       console.log('Certificate verification successful');
   }).on('error', (err) => {
       console.error('Certificate verification failed:', err);
   });
   ```

2. Run the script using Node.js:

   ```bash
   node check_ca_certs.js
   ```

   If the `NODE_EXTRA_CA_CERTS` environment variable is correctly set and Node.js is loading the `bundle.pem` file, the script should output "Certificate verification successful". If there's an issue with the certificates or the loading process, it will output "Certificate verification failed" along with an error message.

This script explicitly sets the global root certificates for the HTTPS module to the ones specified in the `bundle.pem` file, which should trigger the use of your custom CA bundle for certificate verification.
