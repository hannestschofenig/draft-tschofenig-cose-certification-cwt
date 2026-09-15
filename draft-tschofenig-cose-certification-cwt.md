---
title: "Certification CWTs"
abbrev: Certification CWTs
docname: draft-tschofenig-cose-certification-cwt-latest
category: std
submissiontype: IETF
ipr: trust200902
area: Security
workgroup: COSE
keyword: Internet-Draft
stand_alone: yes
pi:
  rfcedstyle: yes
  toc: yes
  tocindent: yes
  sortrefs: yes
  symrefs: yes
  strict: yes
  comments: yes
  inline: yes
  text-list-symbols: -o*+
  docmapping: yes
author:
  -
    ins: H. Tschofenig
    name: Hannes Tschofenig
    organization: University of the Bundeswehr Munich
    abbrev: UniBw M.
    city: Neubiberg
    country: Germany
    code: 85577
    email: hannes.tschofenig@gmx.net
  -
    ins: B. Moran
    name: Brendan Moran
    organization: Arm Limited
    email: brendan.moran.ietf@gmail.com
  -
    name: Henk Birkholz
    org: Fraunhofer SIT
    abbrev: Fraunhofer SIT
    email: henk.birkholz@sit.fraunhofer.de
    street: Rheinstrasse 75
    code: '64295'
    city: Darmstadt
    country: Germany
normative:
  RFC2119:
  RFC3986:
  RFC6838:
  RFC8174:
  RFC8392:
  RFC8610:
  RFC8747:
  RFC8949:
  RFC9052:
  RFC9054:
  RFC9679:
informative:
  RFC5280:
  RFC6024:
  RFC8446:
  RFC8613:
  RFC9019:
  RFC9147:
  RFC9360:
  I-D.ietf-cose-cbor-encoded-cert:
  I-D.ietf-jose-pq-composite-sigs:
  I-D.ietf-oauth-status-list:
--- abstract

This specification defines a profile for using signed CBOR Web Tokens
(CWTs) as COSE-native certification credentials. A Certification CWT binds
a logical subject identity to one public COSE key or to a key set, along
with constraints on certification and key use. Certification CWTs can be
arranged into an ordered certification path rooted in a locally configured
trust anchor.

The profile supports alternative key sets and key sets for which all keys
are required. All-required key sets support protocol-level co-authorization
by separately managed keys, including keys held by independent signing
services or administrative parties. A single key can also use an atomic
composite signature algorithm. A monotonically increasing certification
generation provides supersession and rollback protection without requiring a
trusted real-time clock.

This specification also defines COSE header parameters for carrying and
referencing Certification CWTs and their certification paths.

--- middle

# Introduction

CBOR Web Tokens (CWTs) {{RFC8392}} convey claims protected with CBOR Object
Signing and Encryption (COSE) {{RFC9052}}. The confirmation claim defined in
{{RFC8747}} allows an issuer to bind a CWT to a proof-of-possession key.
Neither specification defines how one signed CWT can authorize the issuer of
another CWT, how certification constraints are propagated, or how a sequence
of such CWTs is validated as a certification path.

This document defines a COSE-native certification credential called a
Certification CWT. A Certification CWT binds a stable logical subject to one
public key or to a key set. It also states whether the subject may issue
further Certification CWTs, limits the remaining certification path, and
constrains the purposes and algorithms for which the subject keys may be used.

This document also defines:

- a certification path validation algorithm;
- alternative and all-required key-set semantics;
- a generation-based supersession and anti-rollback mechanism;
- rules for binding a validated leaf key or key set to a COSE application
  object;
  and
- COSE header parameters for carrying or referencing Certification CWTs.

The trust anchor is an input to path validation. How a device obtains or
stores a trust anchor, for example as a complete public key in ROM, a COSE
Key Thumbprint (CKT) in one-time-programmable memory, or a key in a secure
element, is outside the scope of this specification.

# Conventions and Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in BCP 14
{{RFC2119}} {{RFC8174}} when, and only when, they appear in all capitals,
as shown here.

This document uses the terminology of CWT {{RFC8392}}, CWT confirmation
methods {{RFC8747}}, COSE {{RFC9052}}, and COSE Key Thumbprints {{RFC9679}}.
It additionally defines the following terms:

Certification CWT:
: A signed CWT conforming to the claims and processing rules in this
  document. It binds a subject identity to a public key or key set and to
  certification and key-use constraints.

Certification Path:
: An ordered sequence of one or more Certification CWTs that starts at the
  Target Certification CWT and proceeds toward a locally configured trust
  anchor. The trust anchor is not part of the path.

Issuer:
: The logical authority identified by the iss claim and authorized by the
  current path-validation state to sign the next Certification CWT.

Subject:
: The logical authority identified by the sub claim. A subject is distinct
  from any individual key that represents it.

Key Set:
: One or more public keys that jointly represent one logical authority,
  together with a rule stating which signatures are required.

Alternative Key Set:
: A key set in mode any. A valid signature from any one authorized member is
  sufficient.

All-Required Key Set:
: A key set in mode all. A valid signature from every member is required.

Composite Signature Key:
: One COSE key representing an atomic composite signature algorithm, such as
  a PQ/T algorithm defined by
  {{I-D.ietf-jose-pq-composite-sigs}}. Its component keys and
  signatures are processed inside the composite algorithm and do not form a
  Key Set in this specification.

Trust Anchor:
: A locally trusted authority represented by a public key or key set, an
  authority identifier, initial constraints, and a local algorithm policy.
  This definition follows the architectural concept in {{RFC6024}} and
  {{RFC9019}}.

Target Certification CWT:
: The Certification CWT at the end of the Certification Path whose subject
  key or key set is to be used. This corresponds to the "target certificate"
  terminology used by {{RFC5280}} for certification path validation.

COSE Application Object:
: A COSE object whose signature or use of the sender's static key agreement
  public key is to be authorized using the validated subject key or key set of
  the Target Certification CWT. RFC 5280 does not define a corresponding term
  for this application-layer object.

Certification State:
: The complete subject key or key set and constraints asserted by one
  Certification CWT at a particular certification generation.

# Certification Model

## Trust Anchor Input

Path validation MUST begin with a trust anchor obtained from a trusted local
configuration. The trust-anchor input consists of at least:

- one public COSE key or one key set;
- a logical authority identifier;
- initial certification and key-use constraints; and
- a local algorithm policy.

A key reference in the local representation MUST be resolved to a public key
before it is used. Failure to resolve a required key is a validation failure.

A self-signed CWT MAY be used by a deployment as a representation of a trust
anchor. It is not part of the Certification Path and its presence in an
untrusted message MUST NOT add or replace a trust anchor.

## Certification CWT Protection

A Certification CWT MUST be a COSE_Sign1 or COSE_Sign object. MACed,
encrypted-only, and unsecured CWTs MUST NOT be used as Certification CWTs.
The CWT Claims Set MUST be carried as the embedded COSE payload. Detached
payloads MUST NOT be used. External Additional Authenticated Data MUST be the
zero-length byte string.

A single-key issuer MUST use COSE_Sign1. An issuer represented by an
all-required key set MUST use COSE_Sign so that each required key signs the
same CWT Claims Set. An alternative key-set issuer MAY use COSE_Sign1 or
COSE_Sign.

An issuer represented directly by a Composite Signature Key, rather than by a
Key Set containing that key, is a single-key issuer for this rule. Such an
issuer uses one composite signature in COSE_Sign1; the composite verification
algorithm is responsible for validating all of its components.

The alg and kid parameters used for each signature MUST be carried in the
protected header bucket. Within this profile, kid MUST contain the CKT of the
signing public key as defined by {{RFC9679}}. A recipient MUST reject a
missing, duplicate, or ambiguous mapping between a signature and an issuer key.

When an alternative key-set issuer uses COSE_Sign, every present signature
MUST map to a different authorized member and MUST validate successfully. At
least one signature MUST be present. An application MUST NOT use an invalid or
unmapped additional signature to satisfy mode any.

Tagged and untagged COSE_Sign1 and COSE_Sign objects are allowed when their
type is unambiguous from the surrounding protocol. The optional CWT tag 61
does not alter the claims or signature validation rules.

## Certification CWT Claims

Every Certification CWT MUST contain:

- iss, identifying the issuing logical authority;
- sub, identifying the subject logical authority;
- cti, identifying the Certification State within the issuer context;
- certification-generation;
- exactly one of cnf and subject-key-set; and
- certificate-constraints.

The nbf and exp claims are optional and provide additional time-based validity
when a trusted clock is available.

The iss and sub values MUST be StringOrURI values as defined by {{RFC8392}}.
For path validation they are compared as the exact UTF-8 strings carried in
the CWT. URI normalization, case folding, Unicode normalization, and human
display equivalence MUST NOT be applied. The sub claim identifies a logical
authority and not an individual public key.

The cti value is an opaque byte string. An issuer MUST NOT issue two different
Certification States with the same tuple of iss, sub, and
certification-generation. A recipient that observes the same tuple from the
same validated issuer with a different cti MUST reject the conflicting state.

The following CDDL {{RFC8610}} uses temporary labels pending IANA assignment.
The CWT Claims Set and all newly defined maps MUST be deterministically encoded
as specified in Section 4.2 of {{RFC8949}}. Signature validation operates on
the received payload bytes and MUST NOT re-encode a payload before validating
its signature.

~~~ cddl
Certification_CWT = COSE_Sign1 / COSE_Sign

Certification_Claims = {
  1 => text,                         ; iss
  2 => text,                         ; sub
  7 => bstr,                         ; cti / state identifier
  ? 4 => number,                     ; exp
  ? 5 => number,                     ; nbf
  TBD1 => uint,                      ; certification-generation
  (8 => single-key-confirmation) /   ; cnf
    (TBD2 => subject-key-set),
  TBD3 => certificate-constraints
}

single-key-confirmation = {
  (1 => public-cose-key) /           ; COSE_Key
  (5 => bstr)                        ; ckt
}

public-cose-key = COSE_Key
~~~

Private key parameters MUST NOT occur in public-cose-key. Encrypted_COSE_Key
and the application-specific kid confirmation method from {{RFC8747}} are not
allowed by this profile. When ckt is used, the recipient MUST obtain exactly
one matching public COSE key from a trusted local resolver. The resolved key's
CKT MUST equal the referenced value.

Certification_Claims is a closed profile. An unrecognized claim in a
Certification CWT MUST cause validation to fail unless an application profile
explicitly defines that claim and its validation processing. This rule
overrides the default behavior of ignoring unknown CWT claims for this
specific use of CWT.

# Subject Key Sets

## Key-Set Encoding

The subject-key-set claim binds all keys that represent the subject and the
rule by which they are used. It has the following CDDL:

~~~ cddl
subject-key-set = {
  1 => key-set-mode,
  2 => [+ key-entry],
  ? 3 => bstr                       ; set-id
}

key-set-mode = 1 / 2               ; 1: any, 2: all

key-entry = {
  1 => bstr,                        ; key-id: RFC 9679 CKT
  ? 2 => public-cose-key
}
~~~

The key-id member is mandatory and is the CKT of the represented key. If a
public-cose-key is present, its calculated CKT MUST equal key-id. If the key
is omitted, the recipient MUST resolve key-id to exactly one public key before
the key is required. An unresolved required key causes validation to fail.

Each key-id MUST be unique in the key set. Duplicate public keys, duplicate
key identifiers, an empty key set, and ambiguous key resolution MUST be
rejected. Array order has no authorization semantics. An optional set-id MUST
identify the complete key-set state, including mode and members. How
set-id is generated is to be specified before publication.

## Alternative Key Sets

For mode any, at least one valid signature from an authorized key-set member
is sufficient. A recipient MUST NOT treat the presence of multiple keys as a
requirement for multiple signatures unless mode is all.

An application can use mode any for algorithm transition, key rotation with
an overlap period, or several interchangeable representatives of the same
logical authority.

## All-Required Key Sets

For mode all, one valid signature from every key-set member is required. All
signatures MUST protect the same embedded Certification_Claims payload and the
same external AAD value. Each signature MUST be mapped to a different required
key through its protected kid value.

An all-required key set represents one logical authority under the joint
control of several separately represented keys. It is useful when:

- independent administrative parties must unanimously approve an issuance or
  COSE Application Object, for example a manufacturer and an operator;
- member private keys are controlled by separate signing services or HSMs
  and each signature needs to remain independently identifiable and auditable;
- an application requires a fixed unanimous multi-signature policy using
  already registered COSE algorithms;
- a deployment needs a PQ/T transition before an appropriate atomic composite
  signature suite is standardized and implemented; or
- key-set membership and signer accountability need to be managed at the
  certification-policy layer rather than fixed inside a cryptographic suite.

The subject remains one logical authority. Mode all does not create a separate
Certification Path for every member and does not express a general threshold:
every listed member is required. Removing a key, removing a signature, or
changing mode from all to any MUST cause validation to fail unless a newer,
valid Certification State explicitly authorizes the complete replacement key
set.

This construction is an application rule over multiple COSE signatures. It is
not a cryptographic composite-signature algorithm and does not claim the
security properties of one. In particular, it does not define a cryptographic
signature combiner, a composite domain-separation label, or non-separability
of the component signatures.

When one authority owns a PQ/T key whose two components always have one
lifecycle and every operation requires both components, a composite COSE
signature algorithm such as one specified by
{{I-D.ietf-jose-pq-composite-sigs}} can instead be represented as one public
COSE key and one signature once its COSE algorithm value has been assigned.
Such a composite key follows the single-key rules of this document and uses
COSE_Sign1. Mode all remains appropriate when the separate identity,
auditability, administration, or policy lifecycle of the member keys is an
application requirement. Application profiles MUST state whether they permit
composite algorithms, all-required key sets, or both, and MUST NOT silently
translate between these representations.

## Relationship to Composite Signature Keys

A Composite Signature Key MAY be conveyed by `cnf` as the subject's single
key. It MAY also occur as one member of a `subject-key-set`; in that case the
complete composite key counts as one member, and the key-set mode is applied
outside the composite algorithm. An application profile that permits the
latter construction MUST account for the resulting nested signature
requirements and resource use.

The CKT and protected `kid` identify the complete Composite Signature Key, not
an internal component. Algorithm constraints and local policy identify the
composite COSE algorithm as one algorithm. The internal component
algorithms MUST NOT be substituted for the composite algorithm when evaluating
constraints or satisfying a required signature. The protected `alg` value MUST
match the composite algorithm identifier in the Composite Signature Key.

# Certification Constraints

Every Certification CWT contains certificate-constraints:

~~~ cddl
certificate-constraints = {
  1 => bool,                        ; can-issue
  ? 2 => uint,                      ; path-length
  3 => [+ key-purpose],
  ? 4 => [+ int],                   ; subject COSE algorithms
  ? 5 => [+ key-purpose],           ; delegable key purposes
  ? 6 => [+ int]                    ; delegable COSE algorithms
}

key-purpose = 1 / 2 / 3 / int
; 1: sign Certification CWTs
; 2: sign COSE application objects
; 3: static key agreement
~~~

The can-issue value states whether the subject is allowed to issue another
Certification CWT. A missing value is not allowed. To issue a child
Certification CWT, the current issuer MUST have can-issue true and its direct
key-purpose array MUST contain purpose 1. If either condition is absent, it
MUST NOT issue a child Certification CWT.

Path-length is the maximum number of additional non-leaf Certification CWTs
that may follow this subject in a path. It MUST be present when can-issue is
true and MUST be absent when can-issue is false. A child issuer constraint
MUST be strictly less than the remaining issuer path-length. A leaf has
can-issue false.

The key-purpose array is the complete set of direct generic uses permitted for
the subject key or key set. Application-specific purposes require an
application profile and an IANA assignment or other collision-resistant
identifier space.

The delegable-key-purpose array states which direct purposes the subject is
allowed to grant to descendants. It MUST be present when can-issue is true and
MUST be absent when can-issue is false. A child key-purpose and a child's own
delegable-key-purpose MUST each be subsets of the current issuer's
delegable-key-purpose. This separates use of an issuer's own key from the uses
it may authorize; for example, a certification-only issuer can authorize an
application-signing leaf without using its own key for application signing.

If subject algorithms are present, every algorithm used by the subject MUST
occur in that array and in the local algorithm policy. Delegable algorithms
state which algorithms the subject can authorize for descendants. A child
subject algorithm and a child's own delegable algorithm set MUST each be
subsets of the current issuer's delegable algorithms when that member is
present. Absence of an algorithm member does not override local algorithm
policy.

A Composite Signature Key contributes its composite COSE algorithm identifier
to these checks. Its internal component algorithms are not additional subject
algorithms and do not independently satisfy an allowed or delegable algorithm
constraint.

Unknown members of certificate-constraints MUST cause validation to fail.
Constraints are monotonically narrowed along the path; a child cannot grant
an authority or use that its issuer did not possess.

# Generation-Based Validity

## Generation Scope and Comparison

Every Certification CWT contains a non-negative certification-generation.
The persistent generation scope is:

~~~
(validated-issuer, sub)
~~~

validated-issuer is the stable logical issuer authority established by the
successfully validated path. It is not merely the untrusted iss value and it
is not the thumbprint of the issuer's current key. Consequently, an authorized
rotation of the issuer key or key set does not create a new generation scope.

An implementation with multiple independent trust-anchor stores or policy
contexts MUST additionally include a local validation-context identifier in
its persistent storage key:

~~~
(local-validation-context, validated-issuer, sub)
~~~

The local-validation-context is selected before path validation and is never
taken from an untrusted Certification CWT. It is a local input and not a CWT
claim defined by this specification.

For each scope, a recipient that implements this profile MUST maintain the
greatest successfully committed generation and the corresponding cti.

The comparison rules are:

1. If no generation is stored for the scope, the received generation is a
   candidate initial state.
2. A received generation lower than the stored generation MUST be rejected.
3. A received generation equal to the stored generation is accepted only if
   cti equals the stored cti and all other validation succeeds.
4. A received generation greater than the stored generation is a candidate
   superseding state. Generation jumps are allowed.
5. Integer wraparound MUST NOT be accepted.

A higher generation replaces the complete Certification State for that scope,
including all keys, key-set mode, and constraints. Generations are not
maintained separately for individual members of a key set.

Only an issuer already authorized by the current validated path may advance a
subject generation. A subject cannot authorize its own replacement unless a
separate application profile explicitly grants constrained self-rotation.

## Generation Processing Example

This non-normative example illustrates generation processing for one device.
The recipient selected the local validation context `factory-a` before
processing the Certification Path. Path validation establishes the stable
issuer identity `manufacturer.example/issuer`, and the leaf CWT identifies
the subject `device-42`. The persistent scope is therefore:

~~~
(factory-a, manufacturer.example/issuer, device-42)
~~~

Initially, no state is stored for this scope. The recipient receives a valid
Certification CWT with generation 12, cti `h'1201'`, public key K12, and
constraints permitting COSE application-object signing. It validates the
complete path and a COSE Application Object signed with K12. Only after all
checks succeed does it atomically store `(12, h'1201')` and activate the
complete generation-12 Certification State.

The following inputs then produce these results:

| Stored before | Received generation and cti | Result | Stored after |
|---|---|---|---|
| `(12, h'1201')` | `(11, h'1101')` | Reject as rollback | `(12, h'1201')` |
| `(12, h'1201')` | `(12, h'1201')` | Accept as the same state if all other checks succeed | `(12, h'1201')` |
| `(12, h'1201')` | `(12, h'12ff')` | Reject as a generation conflict | `(12, h'1201')` |
| `(12, h'1201')` | `(15, h'1501')` | Stage as a superseding state; the generation jump is allowed | unchanged until commit |

Suppose generation 15 replaces K12 with K15 and narrows the constraints. The
recipient first verifies that the current validated issuer was authorized to
make that replacement. It then validates the new Certification CWT, the
complete path, the narrowed constraints, and the accompanying COSE
Application Object using K15. If every check and application activation
succeeds, it atomically replaces the complete state and stores
`(15, h'1501')`. A later replay of generation 12 is then rejected.

If validation of the generation-15 path or COSE Application Object fails, or
if the activation transaction fails under a profile that discards staged
state on activation failure, the recipient retains `(12, h'1201')`. It MUST
NOT retain K15 while keeping the generation-12 constraints, or commit
generation 15 while continuing to use K12. An authorized rotation of the
issuer's own key does not change this scope because the stable validated
issuer identity, rather than the issuer key thumbprint, is part of the scope.

## Validation and Atomic Commit

Receiving a higher generation MUST NOT immediately modify persistent state.
The recipient MUST first validate the complete Certification Path, all
required signatures and constraints, and, when the chain is supplied with a
COSE Application Object, the application-object binding and required
application purpose.

All generation and cti changes resulting from one validation operation MUST
then be committed atomically or as part of the application's atomic activation
transaction. A power failure MUST NOT leave a mixture of old and new
generation state. The persistent store itself needs protection against
rollback if an attacker can restore an older storage snapshot.

An implementation MAY stage validated state before application activation,
but MUST define whether an activation failure commits or discards the staged
generations. An application profile SHOULD select one behavior to avoid both
persistent denial of service and rollback windows.

## Time and Status

Certification generation provides supersession and rollback protection; it
does not prove that a credential is recent, and it does not notify a device
that never receives a newer generation.

If nbf or exp is present and the recipient has a trusted clock, the recipient
MUST apply the corresponding RFC 8392 validation rule. An application profile
that requires time-based validity MUST require a trusted clock and state which
time claims are mandatory. A recipient without trusted time MUST NOT treat an
unverified time claim as a successful validity check.

An application profile MAY additionally require an online or periodically
delivered status object, for example the mechanism in
{{I-D.ietf-oauth-status-list}}. The format, freshness, and failure policy for
such status information are outside the scope of this version of the
specification.

A compromised issuer can sign an extremely high generation and thereby cause
a fast-forward denial of service. Recovery requires an authority stronger
than the normal generation-advance authority. A future version or companion
profile is expected to define an authority epoch or equivalent root-authorized
trust-state transition. Implementations MUST NOT reset a stored generation
based only on a normal Certification CWT.

# Certification Path Validation

## Path Construction

The baseline profile accepts one unambiguous path ordered from the Target
Certification CWT toward the trust anchor:

~~~
[ leaf, intermediate-1, intermediate-2 ]
~~~

The trust anchor is not included. The path MAY omit a contiguous suffix of
issuer Certification CWTs that the recipient already has in trusted local
storage. It MUST NOT contain a gap between supplied elements. The baseline
profile does not require discovery among an unordered bag, cross-certification,
or selection among multiple candidate issuers.

An implementation MAY support path building as an extension. If several
valid paths produce different constraints, an application profile MUST define
a deterministic selection rule; otherwise the recipient MUST reject the
ambiguity.

## Validation Inputs

Path validation takes the following inputs:

- one candidate Certification Path;
- one locally configured trust anchor;
- one local validation context selected before processing untrusted input;
- persistent generation and cti state;
- a local algorithm policy;
- locally resolvable keys, if CKT references are used;
- an optional trusted current time; and
- the key purpose required by the caller.

When validating a COSE Application Object, that object and its application
profile are also inputs.

## Validation Procedure

Although the transported path is leaf-to-root, validation proceeds from the
trust anchor toward the leaf. The validator initializes the current authority,
key or key set, constraints, and algorithm policy from the trust anchor. For
every Certification CWT from the trust-anchor end of the path toward the leaf,
it performs all of the following steps:

1. Decode the CWT and enforce configured size, nesting, and operation limits.
2. Verify that it is a permitted COSE_Sign1 or COSE_Sign object with an
   embedded payload and empty external AAD.
3. Determine the signatures required by the current issuer key or key set.
4. Map every present protected kid to exactly one current issuer key, reject
   duplicate or unauthorized mappings, and validate the signatures required by
   the single-key case or key-set mode any or all. For COSE_Sign, every present
   signature MUST validate.
5. Decode Certification_Claims and reject missing, duplicate, malformed, or
   unsupported claims.
6. Compare iss with the current validated authority identifier using the exact
   comparison rules in this document.
7. Verify that the current constraints authorize issuance and that the
   signature algorithms are permitted by both inherited and local policy.
8. Apply nbf and exp when trusted time and the applicable requirement exist.
9. Apply the generation and cti checks without changing persistent state.
10. Resolve and validate the subject key or complete subject key set. Reject
    private key material, duplicate keys, unsupported modes, and unresolved
    required references.
11. Verify that certificate-constraints only narrow inherited authority,
    direct and delegable purposes, path length, and algorithms.
12. Stage the subject identity, key or key set, constraints, generation, and
    cti as the current validation state for the next element.

After the leaf is validated, the validator checks that its key-purpose permits
the use requested by the caller. If a COSE Application Object is supplied, the
application-object binding procedure below also MUST succeed. Only after all
checks succeed may the staged generation state be atomically committed.

Any failed step fails the complete validation operation. An implementation
MUST NOT return a subject key as trusted when path validation fails.

# Binding to a COSE Application Object

The validated leaf subject key or key set is used to authenticate the COSE
Application Object. The application profile MUST identify the required key
purpose and the COSE structure being authenticated.

Every COSE Application Object signature used to satisfy this procedure MUST
carry `alg` and `kid` in its protected header bucket. The protected `kid` MUST
equal the CKT of the complete leaf key that verifies the signature, and `alg`
MUST be permitted by the leaf constraints and local algorithm policy.

For a single key, including a Composite Signature Key, the COSE Application
Object MUST contain at least one valid signature made by that key. For mode
any, at least one valid signature from an authorized leaf key is required. For
mode all, the COSE Application Object MUST be a COSE_Sign object with one valid
signature from every key-set member over the same payload and external AAD.
Each signature's protected `kid` MUST equal the member's CKT.

Additional signatures do not satisfy a missing required signature. An
application profile determines whether additional, non-required signatures
are allowed or cause rejection.

Successful certification-path validation proves only the generic key purpose
and constraints defined here. It does not grant application-specific rights.
The application MUST separately map the local validation context, validated
issuer and subject identities, key-set identity, and purposes to its
authorization policy.

# COSE Header Parameters

The following header parameters carry or reference Certification CWTs. They
are analogous to the X.509 parameters in {{RFC9360}} and the C509 parameters
in {{I-D.ietf-cose-cbor-encoded-cert}}, but contain Certification CWTs and are
processed according to this specification.

~~~ cddl
COSE_CWT = CWT-Messages / [ 2* CWT-Messages ]
CWT-Messages = bstr
COSE_CWTHash = [ hashAlg: int / tstr, hashValue: bstr ]
~~~

CWT-Messages contains the complete serialized Certification CWT, including
any tags that are present. A cwt-t hash is computed over exactly those bytes.

## cwt-chain

cwt-chain contains one Certification CWT or an ordered array of Certification
CWTs. Multiple entries are ordered leaf-to-root as specified in Path
Construction. The trust anchor MUST NOT be included.

The value is untrusted input and MAY be carried in the protected or unprotected
header bucket. Changing an unprotected chain cannot create a valid
Certification CWT, but it can cause denial of service or substitute another
valid credential for the same key. The COSE Application Object MUST bind the
exact leaf Certification CWT by carrying either cwt-chain itself or cwt-t in a
protected header bucket. A protected signature kid identifies a key and is not
by itself a binding to one particular Certification CWT.

## cwt-bag

cwt-bag contains one Certification CWT or an unordered array of Certification
CWTs. It can contain duplicates, unrelated CWTs, and incomplete paths. The
baseline profile does not require support for path building from cwt-bag. An
implementation that supports it MUST apply resource limits and MUST validate
the selected path exactly as specified above.

The presence of a self-signed CWT in cwt-bag MUST NOT modify the configured
trust anchors.

## cwt-t

cwt-t identifies one serialized Certification CWT with COSE_CWTHash. SHA-256
as registered for COSE by {{RFC9054}} MUST be supported. An application MAY
require a different or stronger hash algorithm.

cwt-t is an identifier and does not by itself establish trust. When cwt-t is
used to bind a COSE Application Object to a leaf Certification CWT, it MUST
occur in a protected header bucket. This use is distinct from the CKT
confirmation method in {{RFC9679}}: cwt-t hashes a complete Certification CWT,
while CKT identifies a COSE key.

## cwt-u

cwt-u is a URI {{RFC3986}} referring to one Certification CWT or to a
Certification Path. The referenced resource can use application/cwt or
application/cwt with the usage parameter set to chain.

Retrieval does not establish trust. Every retrieved Certification CWT MUST be
validated to a configured trust anchor. Retrieval implementations MUST set
limits for response size, redirects, nested references, and concurrent fetches.
TLS {{RFC8446}}, DTLS {{RFC9147}}, or OSCORE {{RFC8613}} can protect retrieval,
but transport authentication MUST NOT silently create a new Certification CWT
trust anchor.

## Header Parameter Summary

~~~
+===========+=======+===============+===============================+
| Name      | Label | Value Type    | Description                   |
+===========+=======+===============+===============================+
| cwt-bag   | TBD4  | COSE_CWT      | Unordered Certification CWTs |
+-----------+-------+---------------+-------------------------------+
| cwt-chain | TBD5  | COSE_CWT      | Ordered Certification Path   |
+-----------+-------+---------------+-------------------------------+
| cwt-t     | TBD6  | COSE_CWTHash  | Hash of a Certification CWT  |
+-----------+-------+---------------+-------------------------------+
| cwt-u     | TBD7  | uri           | URI for Certification CWTs   |
+-----------+-------+---------------+-------------------------------+
~~~
{: #fig-parameters title="Certification CWT COSE Header Parameters"}

These parameters can occur in COSE_Sign, COSE_Sign1, COSE_Signature, and
COSE_recipient structures as permitted by the surrounding COSE protocol.

# Application Profile Requirements

An application profile using Certification CWTs MUST specify:

- the trust anchors and local validation contexts it accepts;
- the required leaf key purpose;
- the allowed COSE algorithms and key types;
- whether mode any, mode all, or both are accepted;
- whether Composite Signature Keys are accepted and whether they can occur as
  members of a key set;
- whether CKT references and cwt-bag path building are supported;
- whether nbf, exp, trusted time, or external status is required;
- when staged generation state is committed relative to application
  activation;
- whether additional COSE Application Object signatures are allowed;
- all application-specific authorization mappings; and
- concrete limits for path length, key-set size, message size, fetches, and
  cryptographic operations.

An application profile MUST NOT equate certification-generation with an
application object's own version or sequence number. They are separate
namespaces even when committed in one atomic transaction.

# Examples

The following non-normative diagnostic notation illustrates the claims in a
single-key leaf Certification CWT. Cryptographic values are abbreviated.

~~~ cbor-diag
{
  / iss / 1: "manufacturer.example/root",
  / sub / 2: "manufacturer.example/developer",
  / cti / 7: h'24a1...',
  / certification-generation / TBD1: 12,
  / cnf / 8: {
    / COSE_Key / 1: {
      / kty / 1: 2,
      / alg / 3: -7,
      / crv / -1: 1,
      / x / -2: h'...'
    }
  },
  / certificate-constraints / TBD3: {
    / can-issue / 1: false,
    / key-purpose / 3: [2],
    / subject algorithms / 4: [-7]
  }
}
~~~

The next example illustrates an all-required traditional and post-quantum key
set. The containing Certification CWT is a COSE_Sign object with all signatures
required by its issuer key set.

~~~ cbor-diag
{
  / mode / 1: 2,                    / all /
  / keys / 2: [
    {
      / key-id / 1: h'ckt-classical...',
      / public-cose-key / 2: { / classical COSE_Key / }
    },
    {
      / key-id / 1: h'ckt-pqc...',
      / public-cose-key / 2: { / PQC COSE_Key / }
    }
  ],
  / set-id / 3: h'key-set-state...'
}
~~~

Complete deterministic encodings and positive and negative cryptographic test
vectors are required before publication. They need to cover the single-key
case and the any and all key-set modes; rollback and generation conflict;
constraint broadening; missing signatures; unresolved CKT references; and
COSE Application Object binding.

# Security Considerations

## Trust Anchors and Self-Signed Inputs

All Certification CWTs received through COSE headers or retrieval are
untrusted input until a complete path validates to a locally configured trust
anchor. A self-signed CWT in a chain, bag, or retrieved resource cannot add or
replace a trust anchor. Trust-anchor provisioning and update require a separate
authorized process.

## Identity and Key Misbinding

The exact comparison of issuer and subject identifiers prevents relying on
human-equivalent but byte-distinct names. Issuers are responsible for verifying
control of a subject private key before issuance. The enrollment mechanism is
outside scope, but it MUST provide proof of possession or an equivalent
assurance against borrowed-key identity misbinding.

A CKT identifies key material, not all optional COSE_Key attributes and not the
authority to use that key. Authority comes from the validated Certification
CWT, its constraints, the complete path, and application policy.

## Composite and Key-Set Downgrade

The key-set mode and membership are signed claims. Recipients MUST reject a
missing all-required signature, a duplicate key mapping, and any attempt to
reinterpret all as any. An implementation MUST NOT silently ignore an
unsupported required algorithm. A member can be removed or replaced
only through a newer complete Certification State.

An all-required key set is a protocol-level multi-signature construction and
does not acquire the cryptographic properties of a composite signature. Its
security depends on the interaction of the member algorithms, implementations,
and authorization policy. Domain separation, side channels, and common
implementation dependencies need evaluation by an application profile.

A Composite Signature Key is one key using one composite algorithm. Its
component validation, domain separation, non-separability, downgrade rules,
and component-key reuse restrictions are defined by its algorithm
specification, for example
{{I-D.ietf-jose-pq-composite-sigs}}. A validator MUST NOT accept a
standalone component algorithm when policy or an inherited constraint requires
the composite algorithm.

Changing a subject between a Composite Signature Key and an all-required key
set changes the complete Certification State. Such a change requires a newer
generation authorized by the current issuer and MUST NOT be inferred from the
signatures present on an object.

## Generation, Freeze, and Fast-Forward

Persisted generations prevent replay of an older state after a recipient has
accepted a newer one. They do not prevent an attacker from withholding all new
states from a device. Time validity or fresh status information is needed where
resistance to freeze attacks is required.

A compromised issuer can advance a generation to a very high value. Because
generation jumps support intermittently connected devices, the base mechanism
does not prevent this fast-forward attack. Deployments need a recovery
authority stronger than the normal issuer. Resetting a generation without such
authorization makes rollback protection ineffective.

Generation state MUST be atomically stored and protected against storage
rollback. Implementations need to account for power loss, flash wear, integer
limits, and failures between validation and application activation.

## Algorithm and Cross-Protocol Confusion

Signature algorithms and kid values are protected. The validator applies both
inherited constraints and local algorithm policy. Keys MUST NOT be used for a
purpose absent from key-purpose. Applications SHOULD avoid using the same key
in unrelated protocols unless that use is explicitly authorized and the
protocols provide adequate domain separation.

For a Composite Signature Key, algorithm policy is evaluated against the
composite algorithm identifier. Its internal component keys MUST NOT be reused
as standalone keys or as components of another Composite Signature Key when
the composite algorithm specification prohibits such reuse.

## Untrusted Inputs and Resource Exhaustion

Chains, bags, key sets, COSE headers, and URI responses are attacker-controlled
until validation completes. Implementations MUST bound CBOR nesting, path
length, key-set size, signature count, fetched bytes, redirects, concurrent
requests, and cryptographic operations. Duplicate elements, loops, ambiguous
paths, and unknown critical semantics MUST fail closed.

A Composite Signature appears as one COSE signature but can invoke several
component operations and can contain a large PQ signature. Resource limits
MUST account for its complete key size, signature size, memory use, and number
of internal cryptographic operations.

Network retrieval can create timing or traffic-analysis oracles. A validator
SHOULD avoid fetching attacker-selected resources before inexpensive syntax,
size, and policy checks have succeeded.

## Privacy

Stable subject names, cti values, CKTs, CWT thumbprints, and URIs can be used to
correlate activity. Applications requiring identity protection SHOULD encrypt
the surrounding protocol exchange and avoid globally stable identifiers where
they are not necessary.

# IANA Considerations

This section uses temporary values. IANA is requested to make assignments only
when this document is approved for publication.

## CWT Claims

IANA is requested to add the following entries to the "CBOR Web Token (CWT)
Claims" registry. The final Claim Value Types will reference the CDDL in this
document.

~~~
+==========================+===========+==========================+
| Claim Name               | Claim Key | Claim Value Type         |
+==========================+===========+==========================+
| certification-generation | TBD1      | unsigned integer         |
+--------------------------+-----------+--------------------------+
| subject-key-set          | TBD2      | map                      |
+--------------------------+-----------+--------------------------+
| certificate-constraints  | TBD3      | map                      |
+--------------------------+-----------+--------------------------+
~~~
{: #fig-claims title="Certification CWT Claims"}

## COSE Header Parameters

IANA is requested to add the parameters in {{fig-parameters}} to the "COSE
Header Parameters" registry. The Value Registry field is empty and the
Reference field points to this document.

## Certification CWT Key Purposes

IANA is requested to create a "Certification CWT Key Purposes" registry in
the "CBOR Web Token (CWT)" registry group. Values 1, 2, and 3 are assigned to
"Certification CWT Signing", "COSE Application Object Signing", and "Static
Key Agreement", respectively. The registration policy and ranges remain an
open issue for working-group review.

## Media Type application/cwt

Following the registration procedures in {{RFC6838}}, this document requests
that the application/cwt registration from {{RFC8392}} be updated to add the
optional usage parameter. When usage is chain, the
representation is a CBOR Sequence of single-entry COSE_CWT structures, each
containing one serialized Certification CWT in a byte string. Sequence order
is leaf-to-root.

- Type name: application
- Subtype name: cwt
- Required parameters: N/A
- Optional parameters: usage
- Encoding considerations: binary
- Security considerations: RFC 8392 and this document
- Interoperability considerations: N/A
- Published specification: RFC 8392 and this document
- Applications that use this media type: COSE applications using
  Certification CWTs
- Fragment identifier considerations: N/A
- Additional information: N/A
- Person and email address to contact: iesg@ietf.org
- Intended usage: COMMON
- Restrictions on usage: N/A
- Author: COSE WG
- Change controller: IESG
- Provisional registration: No

When usage is absent, this document assigns no chain-ordering semantics to a
sequence of CWT values.

--- back

# Relationship to C509 and PKIX

CBOR Encoded X.509 Certificates (C509) are a compact CBOR representation of
X.509 certificates. C509 retains X.509 semantics, including names, validity,
extensions, and PKIX path validation. One C509 form is an invertible
re-encoding of a DER certificate and another is signed directly over the
CBOR representation. See {{I-D.ietf-cose-cbor-encoded-cert}}.

This specification does not define a CBOR encoding of X.509 certificates and
does not provide conversion to or from X.509. C509 ought to be used when RFC
5280 semantics or compatibility with existing PKIX infrastructure is required.
This specification is intended for deployments that require a CWT- and
COSE-native model, including COSE keys, explicit key-set semantics,
constrained delegation, and generation-based rollback protection.

The path validation algorithm in this document is not the algorithm in
Section 6 of {{RFC5280}}. Implementations MUST NOT assume that an X.509
extension has an implicit Certification CWT equivalent.

# Acknowledgments

We would like to thank Ken Takayama for his valuable review feedback.
