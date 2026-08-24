# Clear explanation of Rust’s Workspace system

Rust’s Workspace system is incredibly useful for managing multiple related crates in a single project, but it can feel tricky at first. In this post, I’ll explain how it works using practical examples—just like we did with the module system—so you can start using workspaces to organize your projects right away.

Let’s use this file structure to simulate a real-world workspace (we’ll build a simple app with shared utility and model crates):
![[Pasted image 20260412123428.png]]

These are the key things we’ll be able to do with our workspace:
1. Share dependencies between crates (no duplicate downloads!)
2. Import code from one crate into another (e.g., `my_app` uses `utils` and `models`)
3. Build/run all crates with a single command

Let’s dive in!
## Example 1: Setting up the Workspace Root
First, let’s create the root `Cargo.toml`—this is what tells Cargo “this directory is a workspace!”
### Step 1: Create the root `Cargo.toml`
![[Pasted image 20260412123640.png]]
#### Key concepts here:
- The `[workspace]` section declares this as a workspace root.
- `members` lists all the crates in our workspace (Cargo will only recognize these as part of the workspace—no implicit mapping!).
- `resolver = "2"` enables better dependency resolution (avoids some edge cases with shared dependencies).
### What we see vs. what Cargo sees:
- **What we see**: A folder with 3 crates inside `crates/`.
- **What Cargo sees**: A workspace with 3 _explicitly declared_ member crates (it ignores any other folders not in `members`).

## Example 2: Creating the Member Crates
Now let’s set up our three crates. We’ll start with the library crates (`utils` and `models`), then the binary crate (`my_app`).
### Step 1: Create the `models` library crate
First, `crates/models/Cargo.toml`:
![[Pasted image 20260412123803.png]]
Then `crates/models/src/lib.rs` (let’s define a simple `User` model):
![[Pasted image 20260412123823.png]]
#### Key concept:
Just like with modules, we use `pub` to make the `User` struct and its methods public—otherwise, other crates in the workspace can’t use them!
### Step 2: Create the `utils` library crate (that depends on `models`
Now let’s make `utils` depend on `models` (so we can use the `User` struct there).
First, `crates/utils/Cargo.toml`:
![[Pasted image 20260412123903.png]]
Then `crates/utils/src/lib.rs` (a utility to format a user’s info):
![[Pasted image 20260412124032.png]]
#### Key concept:
To depend on another crate in the workspace, we use `path = "../relative/path/to/crate"`—this tells Cargo “use the local crate, not a remote one.”
### Step 3: Create the `my_app` binary crate (that uses both `utils` and `models`)
Finally, let’s make our main app that uses both library crates.
First, `crates/my_app/Cargo.toml`:
![[Pasted image 20260412124106.png]]
Then `crates/my_app/src/main.rs`:
![[Pasted image 20260412124114.png]]
## Example 3: Building and Running the Workspace
Now let’s see how to interact with our workspace!
### Step 1: Build the entire workspace
From the root `my_workspace/` directory, run:
```bash
cargo build
```

This builds _all_ crates in the workspace at once, and **(*shares a single*)** `target/` directory (no duplicate build artifacts!).
### Step 2: Run the `my_app` binary
To run a specific binary crate in the workspace, use `-p` (short for `--package`):
```rust
cargo run -p my_app
```
You should see:
```bash
Alice is 30 years old
```
### Step 3: Share external dependencies (optional but powerful!)
Suppose we want to use the `rand` crate in both `utils` and `my_app`. Instead of adding it to both `Cargo.toml` files, we can add it to the _root_ `Cargo.toml` as a shared dependency:
![[Pasted image 20260412124305.png]]
Then in `crates/utils/Cargo.toml`, we can use the shared dependency like this:
![[Pasted image 20260412124322.png]]
## Summary
Let’s recap the key points of Rust’s Workspace system:

1. **Explicit, not implicit**: You _must_ declare member crates in the root `Cargo.toml`’s `members` list—no automatic detection.
2. **Root `Cargo.toml`**: The `[workspace]` section defines the workspace, and `[workspace.dependencies]` shares external crates.
3. **Local dependencies**: Use `path = "../relative/path"` to depend on other crates in the workspace.
4. **Shared build artifacts**: All crates share a single `target/` directory, saving space and build time.
5. **Run specific crates**: Use `cargo run -p crate_name` to run a binary crate in the workspace.

Workspaces are perfect for large projects, monorepos, or any time you want to split your code into reusable crates without the hassle of managing separate repositories.
Thanks for reading! Now go organize your Rust projects into workspaces—you’ll wonder how you ever lived without them :)