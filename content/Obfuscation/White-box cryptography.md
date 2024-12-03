# Usage with virtualization based obfuscation

Sources:
- [Mesh design pattern: hash-and-decrypt](https://rdist.root.org/2007/04/09/mesh-design-pattern-hash-and-decrypt/) - [Archive](https://web.archive.org/web/20240117042122/https://rdist.root.org/2007/04/09/mesh-design-pattern-hash-and-decrypt/)
- loski2619 on the [Sneaker Development](https://discord.gg/sneakerdev) Discord

> The **white-box** model refers to an extreme attack scenario, in which an adversary has full unrestricted access to a cryptographic implementation. A variety of security goals may be posed, the most fundamental being "unbreakability", requiring that any (bounded) attacker should not be able to extract the secret key hardcoded in the implementation, while at the same time the implementation must be fully functional.

Say you want to protect the following code that will reveal a secret only to those who know the password

```javascript
let usrinpt = input("what is the password?")
if(usrinpt == "42"){
	console.log("the cake is a lie")
} else {
	console.log("denied")
}
```

You put it through your virtual machine obfuscator and it outputs a VM and the following instructions in bytecode

```yaml
STRING "what is the password?"
LOAD 0
STRING "42"
EQUIVALENT
# If statement flow
JUMP_IF_FALSE IF_ELSEBLOCK
STRING "the cake is a lie"
PRINT # Logic to call console.log would be here but its not relevent
JUMP IF_END
LABEL IF_ELSEBLOCK
STRING "denied"
PRINT
LABEL IF_END
STOP_VM
```

Now do you see the issue here? Our secret is in plain text! Sure you might have a complex string decoding algorithm or something but the issue still exists, the attacker will always be able to recover our secret without knowing the password. (this applies to any constant data defined in your code, not just strings)

What if I told you that there is a solution to our problem that would lock everything inside the if statements true condition behind an impenetrable wall? We can use hashing and encryption to do this.

Currently, we are doing something like
```javascript
if(input == "42") {
	run_code()
} else {
	run_code()
}
```

Which allows the attacker to see what the valid password is by looking at what we compare the user input to, by including "42" in the comparison there is no way to hide the password since the attacker must execute the comparison to run the program.

But we can change the code to *not* directly include the comparison
```javascript
if(hash(input) == hashedPassword) {
	run_code()
} else {
	run_code()
}
```

Since "42" is a constant that is never rewritten and immediately gets discarded after usage, that means our compiler definitively knows what its value during runtime will be. The compiler can replace the constant "42" with the output of a hash function with the value as input. Then instead of comparing the user input to the valid key, we can compare the *hash* of the valid key to the *hash* of the user input.

This is all well and good but we have one more issue, the valid key may be hidden now, but the secret is still recoverable by the attacker if they read the source code.

We can fix that by encrypting the code inside the if statement with the correct password as a key, and then during runtime we can try to decrypt the code with the users input (*not* the hash of the input).
```javascript
if(hash(input) == hashedPassword) {
	decrypt_and_run_code(input)
} else {
	run_code()
}
```

Our bytecode now looks something like this
```yaml
STRING "what is the password?"
LOAD 0
CALL_HASH_FUNCTION # Logic to call the hash function would be here but is not relevent
STRING "imtheoutputofahashfunction" # This used to be "42"
EQUIVALENT
DECRYPT_BLOCK startIndex endIndex # Will decrypt the code between the JIF and label
# If statement flow
JUMP_IF_FALSE IF_ELSEBLOCK
ENCRYPTED_INSTRUCTION
ENCRYPTED_INSTRUCTION
ENCRYPTED_INSTRUCTION
LABEL IF_ELSEBLOCK
STRING "denied"
PRINT
LABEL IF_END
STOP_VM
```

Look at that, now the only way for our secret to be known is for the user to know the exact password used to decrypt the code.

There are more protections that are possible like this one but I dunno how to implement em yet