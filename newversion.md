=== Goals
* Combine the forward-secrecy of symmetric-key updating with the "future-secrecy" of an OTR-like Diffie-Hellman ratchet.
** By "future-secrecy" we mean that a point-in-time leak of secret keys to a passive eavesdropper will be "healed" by the introduction of new ECDH keys.
* Detect replay / reorder / deletion of messages
* Allow decryption of out-of-order (delayed) messages with minimal reduction in forward secrecy
* Don't leak metadata in cleartext (such as identities or sequence numbers)

{{{
State
------
Each party stores the following values per conversation in persistent storage:

RK           : 32-byte root key which gets updated by DH ratchet
HKs, HKr     : 32-byte header keys (send and recv versions)
NHKs, NHKr   : 32-byte next header keys (")
CKs, CKr     : 32-byte chain keys (used for forward-secrecy updating)
DHIs, DHIr   : ECDH Identity keys
DHRs, DHRr   : ECDH Ratchet keys
Ns, Nr       : Message numbers (reset to 0 with each new ratchet)
PNs          : Previous message numbers (# of msgs sent under prev ratchet)
ratchet_flag : True if the party will send a new ratchet key in next msg
skipped_HK_MK : A list of stored message keys and their associated header keys
                for "skipped" messages, i.e. messages that have not been
                received despite the reception of more recent messages.
                Entries may be stored with a timestamp, and deleted after a
                certain age.


Key Agreement
--------------
 - Parties exchange identity keys (A,B) and handshake keys (Ah,Ai) and (Bh,Bi)
 - Parties assign themselves "Alice" or "Bob" roles by comparing public keys
 - Parties perform triple-DH with (A,B,Ah,Bh) and derive initial keys:
Alice:
  KDF from triple-DH: RK, HKs=<none>, HKr, NHKs, NHKr, CKs, CKr
  DHIs, DHIr = A, B
  DHRs, DHRr = <none>, Bi
  Ns, Nr = 0, 0
  PNs = 0
Bob:
  KDF from triple-DH: RK, HKr=<none>, HKs, NHKr, NHKs, CKr, CKs
  DHIs, DHIr = B, A
  DHRs, DHRr = Bi, <none>
  Ns, Nr = 0, 0
  PNs = 0


Sending messages
-----------------
Local variables:
  MK  : message key

if DHRs == <none>:
  DHRs = generateECDH()
MK = HASH(CKs || "0")
msg = Enc(HKs, Ns || PNs || DHRs) || Enc(MK, plaintext)
Ns = Ns + 1
CKs = HASH(CKs || "1")
return msg


Receiving messages
-------------------
Local variables:
  MK  : message key
  Np  : Purported message number
  PNp : Purported previous message number
  CKp : Purported new chain key
  DHp : Purported new DHr
  RKp : Purported new root key
  NHKp, HKp : Purported new header keys

Helper functions:
  try_skipped_header_and_message_keys() : Attempt to decrypt the message with skipped-over 
  message keys (and their associated header keys) from persistent storage.

  stage_skipped_header_and_message_keys() : Given a current header key, a current message number, 
  a future message number, and a chain key, calculates and stores all skipped-over message keys
  (if any) in a staging area where they can later be committed, along with their associated 
  header key.  Returns the chain key and message key corresponding to the future message number.

  commit_skipped_header_and_message_keys() : Commits any skipped-over message keys from the 
  staging area to persistent storage (along with their associated header keys).

if (plaintext = try_skipped_header_and_message_keys()):
  return plaintext

if Dec(HKr, header):
  Np = read()
  CKp, MK = stage_skipped_header_and_message_keys(HKr, Nr, Np, CKr)
  if not Dec(MK, ciphertext):
    raise undecryptable
  if bobs_first_message:
    DHRr = read()
    RK = HASH(RK || ECDH(DHRs, DHRr))
    HKs = NHKs
    NHKs, CKs = KDF(RK)
    erase(DHRs)
    bobs_first_message = False
else:
  if not Dec(NHKr, header):
    raise undecryptable()
  Np = read()
  PNp = read()
  DHRp = read()
  stage_skipped_header_and_message_keys(HKr, Nr, PNp, CKr)
  RKp = HASH(RK || ECDH(DHRs, DHRr))
  HKp = NHKr
  NHKp, CKp = KDF(RKp)
  CKp, MK = stage_skipped_header_and_message_keys(HKp, 0, Np, CKp)
  if not Dec(MK, ciphertext):
    raise undecryptable()
  RK = RKp
  HKr = HKp
  NHKr = NHKp
  DHRr = DHRp
  RK = HASH(RK || ECDH(DHRs, DHRr))
  HKs = NHKs
  NHKs, CKs = KDF(RK)
  erase(DHRs)
commit_skipped_header_and_message_keys()
Nr = Np + 1
CKr = CKp
return read()
}}}

=== IPR

The Axolotl specification (this wiki) is hereby placed in the public domain.

=== Acknowledgements

Joint work with Moxie Marlinspike.  Thanks to Adam Langley for discussion and improving the receiving algorithm.