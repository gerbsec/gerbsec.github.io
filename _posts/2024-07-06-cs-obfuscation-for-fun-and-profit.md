---
layout: post
category: blog
---
# c# obfuscation: making your code undetectable (but functional)

## introduction

hey its me gerbsec, back again with more writing. as you're probably not aware, i've recently started getting real deep into opsec and rt again and my next challenge was running all of our awesome c# assemblies in memory. only one issue, they are all signatures to hell and back.... so what do i do? do i run yara64.exe on every single binary and rinse and repeat for hours at a time till i get an undetectable binary? NO! I find a tool that obfuscates c#, update it to make it more practical and functional, and then write about it! 

the tool in question is my fork of [smokeyobfuscator](https://github.com/gerbsec/SmokeyObfuscator). which initially had a gui and some other functionality that i decided was not relevant to rt. the author seems to be a dev or something but his tool is great! 

Original author credit: [TrentonH1ll](https://github.com/TrentonH1ll)

## what is c# obfuscation?

c# obfuscation is the process of transforming your code to make it difficult to understand/detect. this is particularly useful for red teamers since our greatest enemy is static signatures right? right?

### why obfuscate?

1. **security**: protect sensitive logic and algorithms.
2. **opsec**: won't get detected on disk or in memory.
3. **IMPORANT**: this won't help with behavioral but its a start.

## prerequisites

to follow along, you'll need:

- visual studio or any c# ide
- [smokeyobfuscator](https://github.com/gerbsec/SmokeyObfuscator)

## setting up smokeyobfuscator

clone the repository and open it in your ide. let's start with the main function where we load the binaries and execute our obfuscation:

```csharp
private void button2_Click(object sender, EventArgs e)
{
    if (string.IsNullOrEmpty(selectedDirectory))
    {
        MessageBox.Show("please select a directory first.");
        return;
    }

    // get all .exe files from the selected directory
    string[] files = Directory.GetFiles(selectedDirectory, "*.exe");

    foreach (string file in files)
    {
        try
        {
            // load file into memory
            byte[] fileBytes = File.ReadAllBytes(file);

            // load module from memory
            ModuleDefMD module;
            using (var ms = new MemoryStream(fileBytes))
            {
                module = ModuleDefMD.Load(ms);
            }

            // perform obfuscation
            NumberChanger.Process(module);
            Strings.Execute(module);
            ProxyInts.Execute(module);
            HideMethods.Execute(module);

            // save the obfuscated file back to disk
            SaveFile(module, file);
        }
        catch (Exception ex)
        {
            // handle errors
            skippedFiles.Add(file);
            Console.WriteLine($"error processing {file}: {ex.Message}");
        }
    }
}
```

here we simply open a directory for files, rotate through the files and execute our obfuscation techniques. at its current state its a a gui tool. in my next release i'll be pushing out a a command line tool that takes a directory for an argument. 
## protections

now, let's break down the different protections used in smokeyobfuscator.

### hide methods

`hide methods` adds custom attributes and modifies method names to make them harder to understand:

```csharp
public static void Execute(ModuleDef module)
{
    // create a type reference for the 'CompilerGeneratedAttribute'
    TypeRef attrRef = module.CorLibTypes.GetTypeRef("System.Runtime.CompilerServices", "CompilerGeneratedAttribute");
    
    // create a constructor reference for the 'CompilerGeneratedAttribute'
    var ctorRef = new MemberRefUser(module, ".ctor", MethodSig.CreateInstance(module.CorLibTypes.Void), attrRef);
    
    // create a new custom attribute using the constructor reference
    var attr = new CustomAttribute(ctorRef);

    // iterate over all types in the module
    foreach (var type in module.GetTypes())
    {
        // iterate over all methods in each type
        foreach (var method in type.Methods)
        {
            // skip runtime special methods, special methods, and methods named 'Invoke'
            if (method.IsRuntimeSpecialName || method.IsSpecialName || method.Name == "Invoke") continue;
            
            // add the custom attribute to the method
            method.CustomAttributes.Add(attr);
            
            // rename the method to make it less recognizable here im using gerbserv.com but it could literally be anything.
            method.Name = "<gerbserv.com>" + method.Name;
        }
    }
}
```

### number changer

`number changer` obfuscates numeric constants by replacing them with a series of mathematical operations:

```csharp
public static void Process(ModuleDefMD module)
{
    foreach (TypeDef type in module.Types)
    {
        foreach (MethodDef method in type.Methods)
        {
            if (method.Body != null)
            {
                for (int i = 0; i < method.Body.Instructions.Count; i++)
                {
                    Instruction instruction = method.Body.Instructions[i];

                    if (instruction.Operand is int && instruction.IsLdcI4() && instruction.OpCode == OpCodes.Ldc_I4)
                    {
                        List<Instruction> instructions = GenerateInstructions(Convert.ToInt32(instruction.Operand));
                        instruction.OpCode = OpCodes.Nop;

                        foreach (Instruction instr in instructions)
                        {
                            method.Body.Instructions.Insert(i + 1, instr);
                            i++;
                        }
                    }
                }
            }
        }
    }
}
```

### strings

`strings` obfuscates string constants by encoding them:

```csharp
public static void Execute(ModuleDefMD module)
{
    MethodDefUser TTH = new MethodDefUser("gerbserv", MethodSig.CreateStatic(module.CorLibTypes.String, module.CorLibTypes.String), MethodImplAttributes.IL | MethodAttributes.Managed, MethodAttributes.Public | MethodAttributes.Static | MethodAttributes.HideBySig | MethodAttributes.ReuseSlot);
    module.GlobalType.Methods.Add(TTH);
    CilBody body = new CilBody();
    TTH.Body = body;
    body.Instructions.Add(OpCodes.Nop.ToInstruction());
    body.Instructions.Add(OpCodes.Call.ToInstruction(module.Import(typeof(Encoding).GetMethod("get_UTF8", new Type[] { }))));
    body.Instructions.Add(OpCodes.Ldarg_0.ToInstruction());
    body.Instructions.Add(OpCodes.Call.ToInstruction(module.Import(typeof(System.Convert).GetMethod("FromBase64String", new Type[] { typeof(string) }))));
    body.Instructions.Add(OpCodes.Callvirt.ToInstruction(module.Import(typeof(System.Text.Encoding).GetMethod("GetString", new Type[] { typeof(byte[]) }))));
    body.Instructions.Add(OpCodes.Ret.ToInstruction());
    foreach (TypeDef type in module.Types)
    {
        if (type.Name != "Resources" || type.Name != "Settings")
        {
            foreach (MethodDef method in type.Methods)
            {
                if (!method.HasBody) continue;
                for (int i = 0; i < method.Body.Instructions.Count(); i++)
                {
                    if (method.Body.Instructions[i].OpCode == OpCodes.Ldstr)
                    {
                        method.Body.Instructions[i].Operand = Convert.ToBase64String(UTF8Encoding.UTF8.GetBytes(method.Body.Instructions[i].Operand.ToString()));
                        method.Body.Instructions.Insert(i + 1, new Instruction(OpCodes.Call, TTH));
                        i++;
                    }
                }
                method.Body.SimplifyBranches();
                method.Body.OptimizeBranches();
            }
        }
    }
}
```

- **method creation**: a new method named `gerbserv` is created in the global type. this method takes a base64-encoded string as an input and returns the decoded string.
- **cilbody**: a cil (common intermediate language) body is created for the method. this body contains the instructions for decoding a base64 string.
- **instructions**:
    - `OpCodes.Nop`: no operation, just a placeholder.
    - `OpCodes.Call`: calls the `get_UTF8` method of the `Encoding` class to get the utf8 encoding.
    - `OpCodes.Ldarg_0`: loads the first argument (the base64 string) onto the stack.
    - `OpCodes.Call`: calls the `FromBase64String` method of the `Convert` class to decode the base64 string.
    - `OpCodes.Callvirt`: calls the `GetString` method of the `Encoding` class to convert the byte array to a string.
    - `OpCodes.Ret`: returns the decoded string.

Now this actually works for now, but i have plans to aes encrypt instead with a randomly generated key and iv. this ensures that everytime this is generated we have a new hash and a new 
## seeing it in action

before obfuscation, our binary is easily detected by av:

![](assets/images/2024-07-06-cs-obfuscation-for-fun-and-profit-image-1.png)

![](assets/images/2024-07-06-cs-obfuscation-for-fun-and-profit-image-2.png)


after running smokeyobfuscator:

![](assets/images/2024-07-06-cs-obfuscation-for-fun-and-profit-image-3.png)

![](assets/images/2024-07-06-cs-obfuscation-for-fun-and-profit-image-4.png)
## conclusion

you should now have a good understanding of c# obfuscation and how to use smokeyobfuscator to protect your code. feel free to reach out on my socials if you have any questions!

best, gerbsec