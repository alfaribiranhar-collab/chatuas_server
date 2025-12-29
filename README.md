# chatuas_server
# inilah chatuas.
import socket
import threading
from datetime import datetime
import tkinter as tk
from tkinter import scrolledtext

class ChatServerGUI:
    def __init__(self, root):
        self.root = root
        self.root.title("Chat Server - Read-Only Chatroom")
        self.root.geometry("800x600")

        # Scrolled text area for chatroom log
        self.text_area = scrolledtext.ScrolledText(root, height=20, width=80)
        self.text_area.pack(padx=10, pady=10)

        # Start Server button
        self.start_button = tk.Button(root, text="Start Server", command=self.start_server)
        self.start_button.pack(pady=5)

        # Server variables
        self.host = '0.0.0.0'
        self.port = 12346
        self.clients = {}  # {client_id: socket}
        self.client_lock = threading.Lock()
        self.server_socket = None

    def log_message(self, message):
        """Log message to the chatroom display."""
        timestamp = datetime.now().strftime("%H:%M:%S")
        self.text_area.insert(tk.END, f"[{timestamp}] {message}\n")
        self.text_area.see(tk.END)

    def start_server(self):
        """Start the server and begin accepting connections."""
        try:
            self.server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            self.server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
            self.server_socket.bind((self.host, self.port))
            self.server_socket.listen(10)
            self.log_message(f"Server started at {self.host}:{self.port}")
            self.start_button.config(state='disabled')
            threading.Thread(target=self.receive_connections, daemon=True).start()
        except Exception as e:
            self.log_message(f"Error starting server: {e}")

    def receive_connections(self):
        """Accept client connections."""
        while True:
            try:
                client_socket, addr = self.server_socket.accept()
                threading.Thread(target=self.handle_client, args=(client_socket, addr), daemon=True).start()
            except:
                break

    def handle_client(self, client_socket, addr):
        client_id = None
        try:
            client_id = client_socket.recv(1024).decode()
        except:
            pass
        with self.client_lock:
            self.clients[client_id] = client_socket
        self.log_message(f"{client_id} connected from {addr}")

        while True:
                data = client_socket.recv(1024).decode()
                if not data:
                    break

                timestamp = datetime.now().strftime("%H:%M:%S")

                if data == "LIST":
                    # Send list of connected clients
                    with self.client_lock:
                        client_list = ",".join(self.clients.keys())
                    client_socket.send(f"CLIENTS:{client_list}".encode())
                    continue

                # Broadcast
                if data.startswith("ALL:"):
                    message = data[4:]
                    self.broadcast_message(f"[{timestamp}] {client_id} (ALL): {message}", sender_id=client_id)
                    self.log_message(f"{client_id} (ALL): {message}")

                # Private message
                elif data.startswith("TO:"):
                    _, target_id, message = data.split(":", 2)
        _, target_id, message = data.split(":", 2)

              # FORMAT YANG DIMENGERTI CLIENT
        private_msg = f"FROM:{client_id}:{message}"
        self.send_private_message(target_id, private_msg)
        self.log_message(f"{client_id} -> {target_id}: {message}")

            except ValueError:
        error_msg = "ERROR: Format TO:ClientID:Message"
        client_socket.send(error_msg.encode())


        except Exception as e:
            self.log_message(f"Error: {e}")

        finally:
            with self.client_lock:
                if client_id in self.clients:
                    del self.clients[client_id]
            client_socket.close()
            self.log_message(f"{client_id} disconnected")

    def send_private_message(self, target_id, message):
        with self.client_lock:
            
            if target_id in self.clients:
                self.clients[target_id].send(message.encode())
            self.log_message(f"Private message sent to {target_id}: {message}")
        
    def broadcast_message(self, message, sender_id):
        with self.client_lock:
            for cid, sock in self.clients.items():
                if cid != sender_id:
                    sock.send(message.encode())


if __name__ == "__main__":
    root = tk.Tk()
    app = ChatServerGUI(root)
    root.mainloop()

    def send_private_message(self, target_id, message):
        with self.client_lock:
            if target_id in self.clients:
                self.clients[target_id].send(message.encode())
            else:
                print(f"[SERVER] Client {target_id} tidak ditemukan")
