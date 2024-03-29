# Notes

This document contains notes and questions that occur during implementation.

## Crypto
- What curves are officially supported?
    - secp256k1 does not seem to work with chrome, but secp256r1 does
    - It could be a result of the cose encoding of the public key, for example, not including the curve. But the standard doesn't seem to specify how to designate a specific P256 curve

## CTAP2
- Firefox doesn't seem to respond to the INIT response sent by the authenticator. Debuging is difficult in Firefox so it is so far not possible to determine why. Chrome is easier to debug, see `google-chrome --enable-logging --v=1`
- Error messages during INIT are ignored by the client
    - If multiple clients try to connect to the authenticator at the same time, only one will be accepted, and in doing so lock the transaction and all other requests should receive an error message of channel busy. However, the error message is returned on the broadcast channel with no distinguishing values returned, i.e. the nonce. As such, a client cannot determine if the error message was intended for it or, and therefore sensibly just waits to time out. This is not only bad from a UI perspective, but it creates the problem that the user does not necessarily know which instance won that race and therefore which request they are "consenting" to, particularly if the only check is a user presence check

## WebAuthN
- GetAssertion in CTAP is not consistent with GetAssertion in WebAuthN
    - There are a number of inconsistencies between the two approaches, which may create a verification/presence checking vulnerability. See [https://www.w3.org/TR/webauthn-1/#op-get-assertion](https://www.w3.org/TR/webauthn-1/#op-get-assertion)
    - In particular step 7 in WebAuthN requires the list of possible credentials to be presented for the user to select which one to use. Only after the user has selected the appropriate credential is the user verification/presence check performed on that credential. As a result, only one signature is generated - for the selected credential - and only one signature counter is incremented.
    - By contrast CTAP performs the User Verification and Presence check before returning any credentials. Furthermore, it optimistically returns a signature on every credential as they are iterated with getNextAssertion (a call that does not exist in WebAuthN). The problem is that the getNextAssertion doesn't appear to perform any UV or UP checks.
    - CTAP offloads the selection to the client, but pre-signs the assertions. This probably won't stop anything working, the signature check only requires it is higher than previously, but it is still creating an inconsistency and would seem to be a very different set of trust assumptions to WebAuthN, i.e. the client is trusted to discard the assertions it does not need.
    - As to whether there is an attack in, it is not immediately obvious. The RP provides a challenge, which is included in the client data hash, which the authenticator signs, so assuming that cryptographic challenge is not the same between innvocations, the signature on the credential should be useless. However, what if a corrupt client or process able to inject into the bus wants to access a different account at the same RP and can initialise their authentication, thus getting a second challenge that they can include in an alternative client hash. The corrupt client could inject this into the communication, causing the correct authenticator to fail, but potentially revealing the alternative credential through side-channel getNextAssertion. It is relatively far-fetched, but something about the approach in CTAP doesn't feel right.
    - A potential further problem with the signatures being produced is that in theory the counter is incremented as well. As far as I understand an RP is only supposed to check that the counter is larger than before - to test it hasn't been reset/moved. However, one can imagine an RP trusting something with a large discrepency - even if it is positive - with caution. Additionally, if these signatures are being erroneously generated, than an invalid credential on a reset device will eventually become valid despite having never been used for signing in.
    - This raises broader questions about the point of the signature count. It would be easy to fake, so what it is the point?
- Interleaving of getAssertion requests could cause problems
    - getNextAssertion takes no arguments, so there is no way of linking which getAssertion it came from, other than being from the same channel. The standard does not specify how this should be dealth with and under what circumstances a request should be rejected, only the 30 second timer.
- Note on getNextAssertion, there is a slight change between 2.0 and the 2.1 draft with additional steps and an additional variable, but it does not resolve the underlying issue.
- When getNextAssertion is called what checks are performed?
    - getNextAssertion allows iterating through the list of credentials - there appears nothing to stop the client continuing to iterate through the list after a credential has been selected by the user. In particular, I don't think the User Presence check is performed again. As such, the end-user may be unaware about such iterations and collections taking place
- In what scenario does an allow list in GetAssertion contain more than one credential?
    - There is an initial exchange between Client and RP to obtain the allow list. One assumes that in normal circumstances there will be a one-to-one relationship between user and credential. However, if a user registers multiple authenticators with the same account this may not be true.
    - Follow-up question, does the initial request made between the client and RP contain any Authenticator information - I don't think it does, which would mean in the scenario of multiple authenticators registered against one account the allow list will contain multiple credentials. However, if that request is made after the GetInfo call, it could be filtered by Authenticator AAGUID (I don't believe this occurs, as if it did there could be a privacy risk)


- How is User/UserEntity data used?
    - It is not included in the GetAssertion information, so cannot be used for filtering at that point. As such, if a RP does not include an allow list, all credentials for that RP ID (domain) will be returned. As such, all GitHub accounts would be returned if multiple user accounts had been registered and the RP did not provide an Allow list.
    - The problem with the above is that if the Authenticator does not have a display the specification states:
        >Update the response to include the first credential’s publicKeyCredentialUserEntity information and numberOfCredentials. User identifiable information (name, DisplayName, icon) inside publicKeyCredentialUserEntity MUST not be returned if user verification is not done by the authenticator.
    - This delegates the credential selection to the client, i.e. the client shows a UI that will allow the user to pick the relevant credential. However, the User.id is a byte string generated by the client or RP, and is not necessarily known to the user. **How is the user supposed to determine the correct credential when the only information shown a byte id?**
- Who is setting the User Presence check?
    - It defaults to True for CTAP2, but can be set to false by the client/RP. As far as I can tell, this is not a value that the user can override. However, since this check is a "consent" check:
        > user presence: Instructs the authenticator to require user consent to complete the operation.

        This seems to create a problem, since setting UP to False could in theory bypass the consent check? If that is the case it might allow a corrupt RP to iterate through credentials on the same Authenticator to determine related accounts. With a UP I believe the credential checking and interaction with the Authenticator is silent, and as such, could fail in the background, revealing the user.id of related accounts.
- Is a user presence check even a consent check?
    - Can consent be granted without first authenticating the user?

## Firefox
Firefox doesn't support CTAP2 on Linux/Mac, only on Windows by using built in Windows API. On other platforms it falls back to U2F.

## Resident keys
Good explanation of resident keys here, might be something we can expand on
- For example, if an authenticator could be linked with another authenticator by storing its Public Key, it could in theory construct multiple copies of the Resident Key - to be stored on a remote server - but which could be access by multiple authenticators. This would allow an authenticator to add new linked authenticators at any time by retrieving and encrypting new copies. The signature counter might even be able to be homomorphic allowing collective incrementing

## Solo keys
- Start in USB mode ./main -b hidg

## PIN
- The description of the PIN algorithm is quite poor for a proposed standard:
    > "pinToken" is used so that there is minimum burden on the authenticator and platform does not have to not send actual encrypted PIN to the authenticator in normal authenticator usage scenarios.
- They are using AES-CBC with an all zero IV. This is not only not consistent with how AES-CBC should be used, but reduces down to AES-ECB. Not necessarily a security issue, but why specify CBC with a zero IV?
- In changing existing PIN the same shared secret key is used twice with two different messages, but the same IV
    > pinHashEnc: Encrypted first 16 bytes of SHA-256 hash of curPin using sharedSecret: AES256-CBC(sharedSecret, IV=0, LEFT(SHA-256(curPin),16)).

    > newPinEnc: Encrypted "newPin" using sharedSecret: AES256-CBC(sharedSecret, IV=0, newPin).
- The FIDO client library source code appears to have two PIN protocols implemented, despite there only being 1 in the standard. The version 1 in their implementation is not consistent with the version in the standard. This doesn't appear to be the case in the release library
- If a PIN is set on the authenticator it must be provided to make a credential. However, it is not required to access the credential. In particular:
    > If pinAuth parameter is not present and clientPin has been set on the authenticator, set the "uv" bit to 0 in the response.
- Why does this not require the PIN to be provided?
    - It might be because the platform will already have got the pin token, and as such, has proved access. The relying party will get uv = false, which might be ok
- It isn't clear to me how one triggers Chrome to set or change a PIN. Maybe it is intended to be managed entirely outside of WebAuthN, but it is part of CTAP2?
- requireUserVerification 	options.uv or pinAuth/pinProtocol

## User Verification
It seems like user verification is hinged very much on registration. During a registration an RP can request user verification. The GetInfo request to the authenticator will return the authenticator capabilities, one of which is uv (user verification). If the authenticator cannot verify the user in and of itself, i.e. through its own UI it has to set this to False - even if it can handle Client PINs. In such a situation, the client will not present that authenticator to the user and the user will not be able to register with it. The issue comes from the option of "Preferred" for UV on the relying party. I believe this will not reject an authenticator that does not provide the UV flag in its GetInfo.

The problem arises in that CTAP2 uses the UV flag in MakeCredential and GetAssertion responses as PIN checks. As such, the response will come back with UV set to True if the PinAuth was valid. If this is forwarded on to the RP (which it appears to be via the authenticatorData in the assertion), the RP will not be able to distinguish between genuine User Verification and PIN user verification.

"User verification performed, but physical user presence not verified (a typical "smartcard scenario") " how is this possibly up=0, uv=1?

Why is the ordering of PIN check and user verification different between makecredential and getassertion. Surely the PIN check should be performed first
## Token Binding
Token binding is described throughout the WebAuthN standard, albeit as an optional parameter. However, there are some security questions with regards to whether it is important for ensuring security.

More details on token binding and WebAuthN can be found at: [https://fidoalliance.org/fido-technote-the-growing-role-of-token-binding/](https://fidoalliance.org/fido-technote-the-growing-role-of-token-binding/)

The reason this is of interest is that just before it was to become a standard Chrome announced it was going to remove the already present support for Token Binding from Chrome. I believe this has now taken place: [https://groups.google.com/a/chromium.org/g/blink-dev/c/OkdLUyYmY1E](https://groups.google.com/a/chromium.org/g/blink-dev/c/OkdLUyYmY1E)

## Resident keys
The language around resident keys is extremely confusing. For example [https://duo.com/labs/tech-notes/resident-keys-and-the-future-of-webauthn-fido2](https://duo.com/labs/tech-notes/resident-keys-and-the-future-of-webauthn-fido2) talks about resident keys being more like discoverable keys, in that they can be stored on the client, client device, or authenticator. The distinction is primarily what is contained within the Credential ID and how and they are indexd. In the case of a resident key, the credential ID is a 16 byte random ID that uniquely identifies that credential. Credentials are stored indexed according to the URL of the RP. When a client wants to connect to an RP it only needs to provide the RP's URL to the authenticator, for it to be able to get the credential and send it to the RP. In effect, the credential ID becomes the user name and no user name is required.

With a non-resident key the credential source is encrypted and set as the Credential ID, thereby storing it on the server. This is the current normal approach, with very few WebAuthN providers forcing the use of resident keys. For a non-resident key, the authenticator performs the same key generation, then encrypts the credential and sets it to be the credential ID, which is returned to the RP. When getting and assertion the RP must return the appropriate credential ID, which it does via the allow list. As such, it is necessary to first provide a username, so that the appropriate credential ID can be returned. The Authenticator decrypts the ID and performs the assertion.

If the authenticator supports resident keys it can decide to store it on the device by providing a random 16 bytes credential ID. As such, the RP is unaware of how it is stored - although it could guess from the length of the credential ID. I think this is why we are seeing the unusual behaviour with the allow list. The test site is designed for non-resident keys but we are storing a resident key. With non-resident keys I think you should always get a single credential in the allow list, since it is tied to a username - I'm not sure this is in the standard, but it makes sense to be like that. When using a resident key the allow list will be empty because no username will necessarily have been provided, so the challenge remains as to how the user selects the correct credential from all credentials associated with an RP, since user identifiable info cannot be returned if the authenticator doesn't have a screen itself:
    > User identifiable information (name, DisplayName, icon) inside publicKeyCredentialUserEntity MUST not be returned if user verification is not done by the authenticator.

Since it is rare for resident keys to be explicitly supported or used at the moment, I suspect this just has come up as an issue yet.

* if using non-resident keys, I assume the signature counter is stored within the credential source, as in, it gets updated each time and sent back re-encrypted.