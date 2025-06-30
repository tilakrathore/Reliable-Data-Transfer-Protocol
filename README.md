file:///Users/trathore/Desktop/Tilak1/emuser/Tilak%20backup%202/MNC/MNC_projects/CSE4_589_%20PA2%20Description.html




Okay, let's explain the logic for managing multiple logical timers using a single hardware timer in a Selective Repeat ARQ sender, given that you have a pkt_timer timeout[1000]; array to store the state of each logical timer.
The core problem is: How do you know when to retransmit packet X, Y, or Z if you only have one timer that can make a "ding" sound?
The Solution Principle: Event Scheduling and Next Event Prediction Instead of each packet having its own physical timer, you record the absolute time when each packet's logical timer should expire. Your single hardware timer is then set to go off at the earliest of these recorded expiration times.

Here's the step-by-step logic:

Key Data Structure: pkt_timer timeout[1000]
Imagine this array as a calendar where each timeout[i] is a slot for packet i's retransmission schedule.

timeout[i].seqnum: The sequence number i.

timeout[i].expiration_time: The specific future time (e.g., sim_time + RTT_estimate) when packet i should be retransmitted if its ACK hasn't arrived.

timeout[i].is_active: A flag indicating if this slot i currently holds a valid, pending timer.

Logic Breakdown by Event:
1. When a Packet is Sent (A_output() function's role):
Action: When the sender decides to send a new packet (let's say packet with seqnum = S), it needs to start a retransmission timer for it.

Logic:

Store a copy of packet S in the sender's buffer. Mark it as "unacknowledged."

Calculate packet S's retransmission expiration_time: current_sim_time() + RTT_ESTIMATE.

Update the timeout array: Set timeout[S].seqnum = S, timeout[S].expiration_time = calculated_time, and timeout[S].is_active = true.

Crucially, update the single hardware timer: After any change to the set of active logical timers (like adding a new one), you must re-evaluate which active logical timer is going to expire soonest.

Find the pkt_timer entry in the entire timeout array that has the smallest expiration_time among all entries where is_active is true.

Calculate the duration for the hardware timer: (earliest_logical_timer_expiration_time - current_sim_time()).

Stop the currently running hardware timer (if any).

Start the hardware timer with this new duration.

Special case: If the duration is zero or negative (meaning the earliest logical timer is already past due), immediately trigger the A_timerinterrupt() function without actually setting a hardware timer.

2. When an ACK is Received (A_input() function's role):
Action: When the sender receives an ACK for a packet (let's say for seqnum = A), that packet's retransmission timer is no longer needed.

Logic:

Validate the ACK A: Check if it's within the current sending window and if packet A was indeed unacknowledged.

If valid, mark packet A in the sender's buffer as "acknowledged."

Cancel the logical timer for packet A: Set timeout[A].is_active = false.

Advance the sender's window base (s_base): In Selective Repeat, s_base advances whenever s_base itself is acknowledged, and any subsequent packets that are also acknowledged. So, keep incrementing s_base as long as pkt_buffer[s_base].is_acked is true.

Crucially, update the single hardware timer: Similar to sending a packet, removing a logical timer means the next soonest logical timer might now be a different one. You must repeat the re-evaluation process:

Find the pkt_timer entry in the timeout array with the smallest expiration_time among the remaining active logical timers.

Calculate the duration and restart the hardware timer accordingly.

If no logical timers are active, stop the hardware timer completely.

3. When the Single Hardware Timer Fires (A_timerinterrupt() function's role):
Action: This is the "ding!" moment. The hardware timer has reached its set duration.

Logic:

Stop the hardware timer (it has done its job for now).

Get the current_sim_time().

Scan for all expired logical timers: Iterate through the entire timeout array.

For every pkt_timer entry timeout[i] where is_active is true AND expiration_time <= current_sim_time():

This packet i has timed out.

Mark timeout[i].is_active = false (cancel its current timer instance).

Retransmit packet i (send its data to tolayer3).

Reschedule a new timer for packet i: Calculate a new expiration_time for this retransmitted packet (current_sim_time() + RTT_ESTIMATE) and call set_logical_timer(i, new_expiration_time). This ensures that if the retransmission also gets lost, another timer will eventually fire.

Crucially, update the single hardware timer for the next event: After processing all currently expired timers, you need to look for the next logical timer that's due.

Find the pkt_timer entry in the timeout array with the smallest expiration_time among all remaining active logical timers.

If such a timer exists, calculate the duration for the hardware timer (earliest_logical_timer_expiration_time - current_sim_time()).
Start the hardware timer with this duration.

Special case: If the duration is zero or negative (meaning the next earliest logical timer is already past due), immediately trigger A_timerinterrupt() again. This handles situations where multiple logical timers expire at roughly the same time or if time has advanced significantly since the last interrupt.

If no active logical timers remain, stop the hardware timer.

In essence, you're using the timeout[1000] array as a list of "future events." Your single hardware timer's job is simply to trigger an interrupt when the very next event on that list is due. The A_timerinterrupt() then processes all events that have become due and schedules the next earliest event for the hardware timer.
