# rustyregister
Serialize/Desirialize  slices of bytes that represent registers and fields in bits.  

Certainly! Here is a Markdown document that describes the entire process we discussed:

---

# Rust Serialization and Deserialization from JSON Description

## Objective

Implement functions in Rust to serialize and deserialize a structure representing a register of bits into and from a slice of bytes. Each field within this structure can vary in size from 1 to several hundreds of bits. The implementation prioritizes speed over memory usage.

## Example Structure

The `RegisterA` struct is defined as follows:

```rust
struct RegisterA {
    field1: u8,   // Size in bits: 3, Offset: 0
    field2: u16,  // Size in bits: 13, Offset: 3
    field3: u32,  // Size in bits: 23, Offset: 16
    field4: u64,  // Size in bits: 35, Offset: 39
    field5: u128, // Size in bits: 67, Offset: 74
    field6: &[u8],// Size in bits: 479, Offset: 141
}
```

## Requirements

1. **Serialization**:
    - Implement a function to serialize an instance of `RegisterA` into a contiguous sequence of bits stored in a slice of bytes (`&[u8]`).
    - Ensure the function can handle any number of fields and varying bit sizes efficiently.
    - Prioritize speed over memory usage in the implementation.
    - Use appropriate Rust crates for performance optimization (e.g., `bitvec` for bit-level operations, `serde` for serialization).

2. **Deserialization**:
    - Implement a function to deserialize a contiguous sequence of bits stored in a slice of bytes (`&[u8]`) into an instance of `RegisterA`.
    - Ensure accurate extraction of field values based on their defined bit sizes and offsets.
    - Prioritize speed over memory usage in the implementation.

3. **Efficiency**:
    - Optimize the serialization and deserialization processes for speed, even if it requires higher memory usage.
    - Leverage Rust crates such as `nom` for parsing and `serde` for serialization when necessary.

4. **Byte Slice Handling**:
    - Ensure that both serialization and deserialization functions operate on slices of bytes (`&[u8]`), converting between the bit-level representation of the fields and the byte-level storage format.

## JSON Description

The structure is described in a JSON file named `struct_description.json`:

```json
{
    "RegisterA": {
        "fields": [
            {
                "name": "field1",
                "type": "u8",
                "size_in_bits": 3
            },
            {
                "name": "field2",
                "type": "u16",
                "size_in_bits": 13
            },
            {
                "name": "field3",
                "type": "u32",
                "size_in_bits": 23
            },
            {
                "name": "field4",
                "type": "u64",
                "size_in_bits": 35
            },
            {
                "name": "field5",
                "type": "u128",
                "size_in_bits": 67
            },
            {
                "name": "field6",
                "type": "&[u8]",
                "size_in_bits": 479
            }
        ]
    }
}
```

## Implementation

### Dependencies

Add the necessary dependencies to your `Cargo.toml`:

```toml
[dependencies]
bitvec = "0.22"  # Check for the latest version on crates.io
serde = { version = "1.0", features = ["derive"] }
```

### Build Script (`build.rs`)

Create a `build.rs` file to read the JSON file and pass its content to the procedural macro:

```rust
use std::env;
use std::fs;

fn main() {
    let out_dir = env::var("OUT_DIR").unwrap();
    let dest_path = std::path::Path::new(&out_dir).join("struct_description.json");

    fs::copy("struct_description.json", &dest_path).unwrap();
}
```

### Procedural Macro (`src/lib.rs`)

Create a procedural macro to generate the necessary serialization and deserialization functions based on the JSON file:

```rust
use proc_macro::TokenStream;
use quote::quote;
use serde::Deserialize;
use syn::{parse_macro_input, LitStr};

#[derive(Deserialize)]
struct Field {
    name: String,
    r#type: String,
    size_in_bits: u32,
}

#[derive(Deserialize)]
struct StructDescription {
    fields: Vec<Field>,
}

#[proc_macro]
pub fn generate_serialization(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as LitStr);
    let file_path = input.value();
    let json_content = std::fs::read_to_string(file_path).expect("Unable to read JSON file");
    let struct_desc: StructDescription =
        serde_json::from_str(&json_content).expect("Invalid JSON format");

    let struct_name = syn::Ident::new("RegisterA", proc_macro2::Span::call_site());

    let serialize_impl = generate_serialize_impl(&struct_name, &struct_desc.fields);
    let deserialize_impl = generate_deserialize_impl(&struct_name, &struct_desc.fields);

    let output = quote! {
        #serialize_impl
        #deserialize_impl
    };

    TokenStream::from(output)
}

fn generate_serialize_impl(struct_name: &syn::Ident, fields: &[Field]) -> proc_macro2::TokenStream {
    let mut serialize_fields = vec![];

    for field in fields {
        let field_name = syn::Ident::new(&field.name, proc_macro2::Span::call_site());
        let bit_range = match field.name.as_str() {
            "field1" => quote! { bits.extend_from_bitslice(&self.#field_name.view_bits::<Msb0>()[5..]); },
            "field2" => quote! { bits.extend_from_bitslice(&self.#field_name.view_bits::<Msb0>()[3..]); },
            "field3" => quote! { bits.extend_from_bitslice(&self.#field_name.view_bits::<Msb0>()[9..]); },
            "field4" => quote! { bits.extend_from_bitslice(&self.#field_name.view_bits::<Msb0>()[29..]); },
            "field5" => quote! { bits.extend_from_bitslice(&self.#field_name.view_bits::<Msb0>()[61..]); },
            "field6" => quote! {
                let field6_bits = BitSlice::<Msb0, _>::from_slice(self.#field_name).unwrap();
                bits.extend_from_bitslice(&field6_bits[..479]);
            },
            _ => unimplemented!(),
        };

        serialize_fields.push(bit_range);
    }

    quote! {
        impl #struct_name {
            pub fn serialize(&self) -> Vec<u8> {
                let mut bits = bitvec![Msb0, u8;];

                #(#serialize_fields)*

                bits.into_vec()
            }
        }
    }
}

fn generate_deserialize_impl(struct_name: &syn::Ident, fields: &[Field]) -> proc_macro2::TokenStream {
    let mut deserialize_fields = vec![];

    for field in fields {
        let field_name = syn::Ident::new(&field.name, proc_macro2::Span::call_site());
        let field_type: syn::Type = syn::parse_str(&field.r#type).unwrap();
        let bit_range = match field.name.as_str() {
            "field1" => quote! { let #field_name = bits[0..3].load_be::<#field_type>(); },
            "field2" => quote! { let #field_name = bits[3..16].load_be::<#field_type>(); },
            "field3" => quote! { let #field_name = bits[16..39].load_be::<#field_type>(); },
            "field4" => quote! { let #field_name = bits[39..74].load_be::<#field_type>(); },
            "field5" => quote! { let #field_name = bits[74..141].load_be::<#field_type>(); },
            "field6" => quote! {
                let field6_size = 479;
                let field6_bytes = (field6_size + 7) / 8;
                let #field_name = &input[141 / 8..(141 + field6_size + 7) / 8];
            },
            _ => unimplemented!(),
        };

        deserialize_fields.push(bit_range);
    }

    quote! {
        impl #struct_name {
            pub fn deserialize(input: &[u8]) -> Self {
                let bits = BitSlice::<Msb0, _>::from_slice(input).unwrap();

                #(#deserialize_fields)*

                #struct_name {
                    #(#deserialize_fields),*
                }
            }
        }
    }
}
```

### Usage

In your main application code, use the procedural macro to generate the serialization and deserialization functions:

```rust
extern crate proc_macro;
extern crate serde;
extern crate serde_json;
extern crate bitvec;

use generate_serialization::generate_serialization;

generate_serialization!("path/to/struct_description.json");

fn main() {
    let field6_data = vec![0u8; 60]; // Example data for field6
    let register = RegisterA {
        field1: 0b101,
        field2: 0b1010101010101,
        field3: 0b10101010101010101010101,
        field4: 0b101010101010101010101010101010

1010101,
        field5: 0b10101010101010101010101010101010101010101010101010101010101010101,
        field6: &field6_data,
    };

    let serialized = register.serialize();
    println!("Serialized data: {:?}", serialized);

    let deserialized = RegisterA::deserialize(&serialized);
    println!("Deserialized struct: {:?}", deserialized);
}
```

### Summary

This document describes how to implement Rust functions to serialize and deserialize a structure based on a JSON description. The process involves creating a JSON file, writing a build script to read the JSON file, and implementing a procedural macro to generate the necessary code. The provided example demonstrates the usage of the generated serialization and deserialization functions.