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