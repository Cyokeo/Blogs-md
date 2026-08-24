
```rust
ast_enum_of_structs! {
/// The possible types that a Rust value could have.
///
/// # Syntax tree enum
///
/// This type is a [syntax tree enum].
///
/// [syntax tree enum]: crate::expr::Expr#syntax-tree-enums
#[cfg_attr(docsrs, doc(cfg(any(feature = "full", feature = "derive"))))]
#[non_exhaustive]
pub enum Type {
/// A fixed size array type: `[T; n]`.
Array(TypeArray),
/// A bare function type: `fn(usize) -> bool`.
BareFn(TypeBareFn),  
/// A type contained within invisible delimiters.
Group(TypeGroup),  
/// An `impl Bound1 + Bound2 + Bound3` type where `Bound` is a trait or
/// a lifetime.
ImplTrait(TypeImplTrait),
/// Indication that a type should be inferred by the compiler: `_`.
Infer(TypeInfer),
/// A macro in the type position.
Macro(TypeMacro),
/// The never type: `!`.
Never(TypeNever),
/// A parenthesized type equivalent to the inner type.
Paren(TypeParen),
/// A path like `core::slice::Iter`, optionally qualified with a
/// self-type as in `<Vec<T> as SomeTrait>::Associated`.
Path(TypePath),
/// A raw pointer type: `*const T` or `*mut T`.
Ptr(TypePtr),
/// A reference type: `&'a T` or `&'a mut T`.
Reference(TypeReference),
/// A dynamically sized slice type: `[T]`.
Slice(TypeSlice),
/// A trait object type `dyn Bound1 + Bound2 + Bound3` where `Bound` is a
/// trait or a lifetime.
TraitObject(TypeTraitObject),
/// A tuple type: `(A, B, C, String)`.
Tuple(TypeTuple),
/// Tokens in type position not interpreted by Syn.
Verbatim(TokenStream),

// For testing exhaustiveness in downstream code, use the following idiom:
//
// match ty {
// #![cfg_attr(test, deny(non_exhaustive_omitted_patterns))]
//
// Type::Array(ty) => {...}
// Type::BareFn(ty) => {...}
// ...
// Type::Verbatim(ty) => {...}
//
// _ => { /* some sane fallback */ }
// }
//
// This way we fail your tests but don't break your library when adding
// a variant. You will be notified by a test failure when a variant is
// added, so that you can add code to handle it, but your library will
// continue to compile and work for downstream users in the interim.
}
}
```