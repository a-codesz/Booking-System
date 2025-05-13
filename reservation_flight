import os
import sys
import random
from collections import deque
import tkinter as tk
from tkinter import ttk, messagebox
from multiprocessing import Process, Pipe, Queue
import json
import time

class Flight:
    def __init__(self, flight_id, source, destination, total_seats, price):
        self.flight_id = flight_id
        self.source = source
        self.destination = destination
        self.total_seats = total_seats
        self.available_seats = total_seats
        self.price = price

    def to_dict(self):
        return {
            'flight_id': self.flight_id,
            'source': self.source,
            'destination': self.destination,
            'total_seats': self.total_seats,
            'available_seats': self.available_seats,
            'price': self.price
        }

class Booking:
    def __init__(self, booking_id, flight_id, passenger_name, seats, priority):  
        self.booking_id = booking_id
        self.flight_id = flight_id
        self.passenger_name = passenger_name
        self.seats = seats
        self.priority = priority

    def to_dict(self):
        return {
            'booking_id': self.booking_id,
            'flight_id': self.flight_id,
            'passenger_name': self.passenger_name,
            'seats': self.seats,
            'priority': self.priority
        }

def booking_processor(pipe_conn, booking_queue):
    """
    Separate process for handling bookings
    """
    flights = {
        'FL001': Flight('FL001', 'New York', 'London', 4, 500.0),
        'FL002': Flight('FL002', 'London', 'Paris', 10, 300.0),
        'FL003': Flight('FL003', 'Paris', 'Tokyo', 8, 800.0)
    }
    bookings = {}

    while True:
        try:
            # Wait for commands from the main process
            command = pipe_conn.recv()

            if command['type'] == 'BOOK':
                booking_data = command['data']
                flight_id = booking_data['flight_id']

                if flight_id in flights:
                    flight = flights[flight_id]
                    if flight.available_seats >= booking_data['seats']:
                        flight.available_seats -= booking_data['seats']
                        bookings[booking_data['booking_id']] = booking_data
                        response = {
                            'status': 'SUCCESS',
                            'booking_id': booking_data['booking_id'],
                            'flight': flight.to_dict()
                        }
                    else:
                        response = {
                            'status': 'ERROR',
                            'message': 'Insufficient seats'
                        }
                else:
                    response = {
                        'status': 'ERROR',
                        'message': 'Flight not found'
                    }

                pipe_conn.send(response)

            elif command['type'] == 'CANCEL':
                booking_id = command['booking_id']
                if booking_id in bookings:
                    booking = bookings[booking_id]
                    flight = flights[booking['flight_id']]
                    flight.available_seats += booking['seats']
                    del bookings[booking_id]
                    response = {
                        'status': 'SUCCESS',
                        'flight': flight.to_dict()
                    }
                else:
                    response = {
                        'status': 'ERROR',
                        'message': 'Booking not found'
                    }
                pipe_conn.send(response)

            elif command['type'] == 'GET_FLIGHTS':
                available_flights = [flight.to_dict() for flight in flights.values()
                                  if flight.available_seats > 0]
                pipe_conn.send({
                    'status': 'SUCCESS',
                    'flights': available_flights
                })

            elif command['type'] == 'GET_BOOKING':
                booking_id = command['booking_id']
                booking = bookings.get(booking_id)
                if booking:
                    response = {
                        'status': 'SUCCESS',
                        'booking': booking
                    }
                else:
                    response = {
                        'status': 'ERROR',
                        'message': 'Booking not found'
                    }
                pipe_conn.send(response)

            elif command['type'] == 'EXIT':
                break

        except EOFError:
            break

    pipe_conn.close()

class FlightReservationSystem:
    def __init__(self):
        
        self.parent_conn, self.child_conn = Pipe()
        self.booking_queue = Queue()

        
        self.processor = Process(
            target=booking_processor,
            args=(self.child_conn, self.booking_queue)
        )
        self.processor.start()

    def list_available_flights(self):
        self.parent_conn.send({'type': 'GET_FLIGHTS'})
        response = self.parent_conn.recv()
        if response['status'] == 'SUCCESS':
             flights = []
        for flight_data in response['flights']:
           
            flight = Flight(
                flight_id=flight_data['flight_id'],
                source=flight_data['source'],
                destination=flight_data['destination'],
                total_seats=flight_data['total_seats'],
                price=flight_data['price']
            )
            flight.available_seats = flight_data['available_seats']  
            flights.append(flight)
        return flights
        return []

    def book_flight(self, flight_id, passenger_name, seats, priority):
        booking_id = f"BK{random.randint(1000, 9999)}"
        booking = Booking(booking_id, flight_id, passenger_name, seats, priority)

        self.parent_conn.send({
            'type': 'BOOK',
            'data': booking.to_dict()
        })

        response = self.parent_conn.recv()
        if response['status'] == 'SUCCESS':
            return response['booking_id']
        return None

    def cancel_booking(self, booking_id):
        self.parent_conn.send({
            'type': 'CANCEL',
            'booking_id': booking_id
        })

        response = self.parent_conn.recv()
        return response['status'] == 'SUCCESS'

    def get_booking_details(self, booking_id):
        self.parent_conn.send({
            'type': 'GET_BOOKING',
            'booking_id': booking_id
        })

        response = self.parent_conn.recv()
        if response['status'] == 'SUCCESS':
            return Booking(**response['booking'])
        return None

    def shutdown(self):
        self.parent_conn.send({'type': 'EXIT'})
        self.processor.join()
        self.parent_conn.close()

class FlightReservationGUI:
    def __init__(self, root):
        self.root = root
        self.root.title("Flight Reservation System")
        self.system = FlightReservationSystem()

        
        self.root.protocol("WM_DELETE_WINDOW", self.on_closing)

        
        self.notebook = ttk.Notebook(root)
        self.notebook.pack(pady=10, expand=True)

        self.flights_tab = ttk.Frame(self.notebook)
        self.booking_tab = ttk.Frame(self.notebook)
        self.manage_tab = ttk.Frame(self.notebook)

        self.notebook.add(self.flights_tab, text="Available Flights")
        self.notebook.add(self.booking_tab, text="Book Flight")
        self.notebook.add(self.manage_tab, text="Manage Booking")

        self._setup_flights_tab()
        self._setup_booking_tab()
        self._setup_manage_tab()

    def on_closing(self):
        self.system.shutdown()
        self.root.destroy()

    
    def _setup_flights_tab(self):
        columns = ('Flight ID', 'Source', 'Destination', 'Available Seats', 'Price')
        self.flights_tree = ttk.Treeview(self.flights_tab, columns=columns, show='headings')

        for col in columns:
            self.flights_tree.heading(col, text=col)
            self.flights_tree.column(col, width=100)

        self.flights_tree.pack(pady=10, padx=10, expand=True, fill='both')
        ttk.Button(self.flights_tab, text="Refresh", command=self._refresh_flights).pack(pady=5)
        self._refresh_flights()

    def _setup_booking_tab(self):
        form_frame = ttk.Frame(self.booking_tab)
        form_frame.pack(pady=20, padx=20)

        ttk.Label(form_frame, text="Flight ID:").grid(row=0, column=0, pady=5, padx=5)
        self.flight_id_var = tk.StringVar()
        self.flight_id_entry = ttk.Entry(form_frame, textvariable=self.flight_id_var)
        self.flight_id_entry.grid(row=0, column=1, pady=5, padx=5)

        ttk.Label(form_frame, text="Passenger Name:").grid(row=1, column=0, pady=5, padx=5)
        self.passenger_var = tk.StringVar()
        self.passenger_entry = ttk.Entry(form_frame, textvariable=self.passenger_var)
        self.passenger_entry.grid(row=1, column=1, pady=5, padx=5)

        ttk.Label(form_frame, text="Number of Seats:").grid(row=2, column=0, pady=5, padx=5)
        self.seats_var = tk.StringVar()
        self.seats_entry = ttk.Entry(form_frame, textvariable=self.seats_var)
        self.seats_entry.grid(row=2, column=1, pady=5, padx=5)

        ttk.Label(form_frame, text="Priority:").grid(row=3, column=0, pady=5, padx=5)
        self.priority_var = tk.StringVar(value="3")
        priority_frame = ttk.Frame(form_frame)
        priority_frame.grid(row=3, column=1, pady=5, padx=5)
        ttk.Radiobutton(priority_frame, text="First Class", variable=self.priority_var, value="1").pack(side=tk.LEFT)
        ttk.Radiobutton(priority_frame, text="Business", variable=self.priority_var, value="2").pack(side=tk.LEFT)
        ttk.Radiobutton(priority_frame, text="Economy", variable=self.priority_var, value="3").pack(side=tk.LEFT)

        ttk.Button(form_frame, text="Book Flight", command=self._book_flight).grid(row=4, column=0, columnspan=2, pady=20)

    def _setup_manage_tab(self):
        form_frame = ttk.Frame(self.manage_tab)
        form_frame.pack(pady=20, padx=20)

        ttk.Label(form_frame, text="Booking ID:").grid(row=0, column=0, pady=5, padx=5)
        self.booking_id_var = tk.StringVar()
        self.booking_id_entry = ttk.Entry(form_frame, textvariable=self.booking_id_var)
        self.booking_id_entry.grid(row=0, column=1, pady=5, padx=5)

        button_frame = ttk.Frame(form_frame)
        button_frame.grid(row=1, column=0, columnspan=2, pady=10)
        ttk.Button(button_frame, text="View Details", command=self._view_booking).pack(side=tk.LEFT, padx=5)
        ttk.Button(button_frame, text="Cancel Booking", command=self._cancel_booking).pack(side=tk.LEFT, padx=5)

    def _refresh_flights(self):
        for item in self.flights_tree.get_children():
            self.flights_tree.delete(item)

        for flight in self.system.list_available_flights():
            self.flights_tree.insert('', 'end', values=(
                flight.flight_id,
                flight.source,
                flight.destination,
                flight.available_seats,
                f"${flight.price:.2f}"
            ))

    def _book_flight(self):
        try:
            flight_id = self.flight_id_var.get()
            passenger_name = self.passenger_var.get()
            seats = int(self.seats_var.get())
            priority = int(self.priority_var.get())

            booking_id = self.system.book_flight(flight_id, passenger_name, seats, priority)
            if booking_id:
                messagebox.showinfo("Success", f"Booking confirmed. Booking ID: {booking_id}")
                self._refresh_flights()
            else:
                messagebox.showerror("Error", "Unable to complete booking")
        except ValueError:
            messagebox.showerror("Error", "Please enter valid numbers")

    def _view_booking(self):
        booking_id = self.booking_id_var.get()
        booking = self.system.get_booking_details(booking_id)
        if booking:
            messagebox.showinfo("Booking Details",
                               f"Booking ID: {booking.booking_id}\n"
                               f"Flight: {booking.flight_id}\n"
                               f"Passenger: {booking.passenger_name}\n"
                               f"Seats: {booking.seats}")
        else:
            messagebox.showerror("Error", "Booking not found")

    def _cancel_booking(self):
        booking_id = self.booking_id_var.get()
        if self.system.cancel_booking(booking_id):
            messagebox.showinfo("Success", "Booking cancelled successfully")
            self._refresh_flights()
        else:
            messagebox.showerror("Error", "Unable to cancel booking")

def main():
    root = tk.Tk()
    root.geometry("800x600")
    app = FlightReservationGUI(root)
    root.mainloop()

if __name__ == "__main__":
    main()

