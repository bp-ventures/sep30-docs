# BP Ventures SEP-30 Documentation

This repository contains documentation for BP Ventures SEP-30 implementation.  
Stellar has [a reference implementation in Golang](https://github.com/stellar/go/tree/master/exp/services/recoverysigner),
but it's tightly integrated with Firebase and Twilio. BPV's SEP-30
implementation is more flexible and allows many types of authentication,
including Email, GitHub, and other social providers.

- [SEP-30 Specification 🔗](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0030.md)
- [Registration Flow](#registration-flow)
- [Recovery Flow](#recovery-flow)
- [Playground 🔗](https://sep30-demo.bpventures.us/)

## Registration Flow

This flow assumes:
- A Stellar Wallet compatible with SEP-30 and pre-configured to use two BPV SEP-30 recovery servers
- Two BPV SEP-30 recovery servers deployed:
  - First one requires Email authentication
  - Second one requires GitHub authentication

```mermaid
sequenceDiagram
    participant U as User
    participant W as Stellar Wallet
    participant RS1 as Recovery Server (Email)
    participant RS2 as Recovery Server (GitHub)
    participant S as Stellar Blockchain
    participant SS as Sponsor Server
    U->>W: Clicks "New Wallet"
    U->>W: Clicks "Register"
    Note over W: Now, we need to register in the first<br>recovery server, which<br>requires email authentication.
    W->>RS1: POST /authorize/token
    RS1-->>W: id + token
    W->>RS1: Open popup pointing to /authorize/?token={token}
    W->>RS1: (loop every 1s) GET /authorize/token/{id}
    alt User not registered
    U->>RS1: Click "Sign-up"
    U->>RS1: Fill email+password & Submit form
    Note over RS1: Confirmation email has been sent
    U->>RS1: Close page
    U->>RS1: Clicks link sent to email
    U->>RS1: Click "Confirm email"
    Note over RS1: Email confirmed
    U->>RS1: Close page
    U->>W: Clicks "Register via Email"
    W->>RS1: Open popup pointing to /authorize/?token={token}
    U->>RS1: Click "Sign-in"
    U->>RS1: Fill email+password & Submit form
    U->>RS1: Click "Authorize"
    Note over RS1: Redirect to /account/logged-in<br>and inform user he can close the page.
    end
    alt User not logged in
    U->>RS1: Click "Sign-in"
    U->>RS1: Fill email+password & Submit form
    U->>RS1: Click "Authorize"
    Note over RS1: Redirect to /account/logged-in<br>and inform user he can close the page.
    end
    alt User logged in
    U->>RS1: Click "Authorize"
    Note over RS1: Redirect to /account/logged-in<br>and inform user he can close the page.
    end
    Note over W: At this point, the "GET /authorize/token/{id}"<br>will contain access_token (let's call it access_token_1),<br>which is a<br>JWT with some information
    Note over W: Stop calling GET /authorize/token/{id}
    Note over W: Now, we need to register in the second<br>recovery server, which<br>requires GitHub authentication.
    W->>RS2: POST /authorize/token
    RS2-->>W: id + token
    W->>RS2: Open popup pointing to /authorize/?token={token}
    W->>RS2: (loop every 1s) GET /authorize/token/{id}
    alt User not registered
    U->>RS2: Click "Sign-in with GitHub"
    U->>RS2: Authorize GitHub access
    Note over RS2: Redirect to /account/logged-in<br>and inform user he can close the page.
    end
    alt User not logged in
    U->>RS2: Click "Sign-in with GitHub"
    U->>RS2: Authorize GitHub access
    Note over RS2: Redirect to /account/logged-in<br>and inform user he can close the page.
    end
    alt User logged in
    U->>RS2: Authorize GitHub access
    Note over RS2: Redirect to /account/logged-in<br>and inform user he can close the page.
    end
    Note over W: At this point, the "GET /authorize/token/{id}"<br>will contain access_token (let's call it access_token_2),<br>which is a<br>JWT with some information
    Note over W: Stop calling GET /authorize/token/{id}
    W->>SS: POST /sponsor/submit
    SS-->>W: Signed envelope to sponsor the account
    W->>S: Submit transaction
    Note over W: Account is sponsored (funded)
    Note over W: Now we need to get SEP-10 tokens from both recovery servers
    W->>RS1: Get SEP-10 token using /sep10/auth
    W->>RS2: Get SEP-10 token using /sep10/auth
    Note over W: Now, we check if the account is already<br>registered in the first recovery server
    W->>RS1: GET /accounts/{address}<br>Bearer {sep10 token}
    alt Account not found in first recovery server
        Note over W: Account not found in first recovery server,<br>this is expected, so we check<br>if the account is already registered in the second<br>recovery server
        W->>RS2: GET /accounts/{address}<br>Bearer {sep10 token}
        alt Account not found in second recovery server
            Note over W: Account not found in second recovery server,<br>this is expected, so we proceed<br>to register it in both servers.
            W->>RS1: POST /sep30/accounts/{address}<br>Bearer {access_token_1}<br>Auth method type: access_token_1.auth_method_type<br>Auth method value: access_token_1.auth_method_value
            RS1-->>W: Account info, including generated signer pubkey
            W->>RS2: POST /sep30/accounts/{address}<br>Bearer {access_token_2}<br>Auth method type: access_token_2.auth_method_type<br>Auth method value: access_token_2.auth_method_value
            RS2-->>W: Account info, including generated signer pubkey
            W->>S: Submit Set-Options transaction and add<br>the signers obtained from both recovery servers
        end
        alt Account found in first recovery server
            Note over W: If we get here it means the account is not registered<br>in the first recovery server,<br>but is registered in the second recovery server.<br>In this case we can't proceed, so we<br>inform the user he needs to recovery the account<br>.
        end
    end
    alt Account found in first recovery server
        Note over W: Account found in first recovery server,<br>so we check if the account is registered<br>in the second recovery server.
        W->>RS2: GET /accounts/{address}<br>Bearer {sep10 token}
        alt Account found in second recovery server
            Note over W: If we get here, it's all as expected,<br>account is registered in both recovery servers.
            W->>U: Display signer info
        end
        alt Account not found in second recovery server
            Note over W: If we get here it means the account is registered<br>in the first recovery server,<br>but is not registered in the second recovery server.<br>In this case we can't proceed, so we<br>inform the user he needs to recover the account<br>.
        end
    end
```

## Recovery Flow

This flow assumes:
- A Stellar Wallet compatible with SEP-30 and pre-configured to use two BPV SEP-30 recovery servers
- Two BPV SEP-30 recovery servers deployed:
  - First one requires Email authentication
  - Second one requires GitHub authentication
- User is already registered in both recovery servers and wants to recover a given pubkey

```mermaid
sequenceDiagram
    participant U as User
    participant W as Retail Wallet
    participant RS1 as Recovery Server (Email)
    participant RS2 as Recovery Server (GitHub)
    participant S as Stellar Blockchain
    U->>W: Click "Recover account"
    U->>W: Enter public key
    Note over W: First, we check if the account is registered<br>in the first recovery server
    W->>RS1: Get a SEP-10 token using /sep10/auth
    W->>RS1: GET /accounts/{address}<br>Bearer {sep10 token of first recovery server}
    alt Account found
        Note over W: Account is registered in first<br>recovery server, now we check<br>if it's also registered in the second<br>recovery server
        W->>RS2: Get a SEP-10 token using /sep10/auth
        W->>RS2: GET /accounts/{address}<br>Bearer {sep10 token of second recovery server}
        alt Account found
            Note over W: Account is registered in both<br>recovery servers, so we can proceed<br>to recover it.
            Note over W: Generate a new Stellar keypair<br>to be the device key
            Note over W Build Set-Options envelope,<br>that sets the signers, weights and thresholds<br>
            W->>RS1: POST /sep30/accounts/{address}/sign/{signer of first recovery server (see registration flow)}<br>Bearer {access_token_1}<br>Auth method type: access_token_1.auth_method_type<br>Auth method value: access_token_1.auth_method_value
            RS1-->>W: Signature
            W->>RS2: POST /sep30/accounts/{address}/sign/{signer of second recovery server (see registration flow)}<br>Bearer {access_token_2}<br>Auth method type: access_token_2.auth_method_type<br>Auth method value: access_token_2.auth_method_value
            RS2-->>W: Signature
            Note over W: Add signatures to the Set-Options envelope
            W->>S: Submit Set-Options transaction
            Note over W: Account has been recovered
        end
        alt Account not found
            Note over W: Account is not registered in the second<br>recovery server, so we can't proceed<br>to recover it.
        end
    end
    alt Account not found
        Note over W: Account is not registered in the first<br>recovery server, so we can't proceed<br>to recover it.
    end
```