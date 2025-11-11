# ELECTION-E--VOTING-APP

# e_voting_streamlit.py
# Streamlit GUI for Secure E-Voting (blockchain-inspired ledger + linked list)
# Kept UI exactly as provided by you. Only change: password prompt now asks each time Admin/Ledger buttons are clicked.

import streamlit as st
import hashlib, json, os
from datetime import datetime

SAVE_FILE = "election_state_streamlit.json"

st.set_page_config(page_title="Secure E-Voting (Streamlit)", layout="wide")

# -------------------------
# Core data classes (same logic)
# -------------------------
class Block:
    def __init__(self, index: int, voter_hash: str, candidate_id: str, timestamp: str, prev_hash: str):
        self.index = index
        self.voter_hash = voter_hash
        self.candidate_id = candidate_id
        self.timestamp = timestamp
        self.prev_hash = prev_hash
        self.hash = self.compute_hash()
        self.next = None

    def compute_hash(self) -> str:
        data = f"{self.index}|{self.voter_hash}|{self.candidate_id}|{self.timestamp}|{self.prev_hash}"
        return hashlib.sha256(data.encode('utf-8')).hexdigest()

    def to_dict(self):
        return {
            "index": self.index,
            "voter_hash": self.voter_hash,
            "candidate_id": self.candidate_id,
            "timestamp": self.timestamp,
            "prev_hash": self.prev_hash,
            "hash": self.hash
        }

    @staticmethod
    def from_dict(d):
        b = Block(d["index"], d["voter_hash"], d["candidate_id"], d["timestamp"], d["prev_hash"])
        b.hash = d["hash"]
        return b

class Ledger:
    def __init__(self):
        self.head = None
        self.tail = None
        self.size = 0

    def append_block(self, block: Block):
        if not self.head:
            self.head = self.tail = block
        else:
            self.tail.next = block
            self.tail = block
        self.size += 1

    def get_block_by_index(self, idx: int):
        cur = self.head
        while cur:
            if cur.index == idx:
                return cur
            cur = cur.next
        return None

    def verify_chain(self):
        cur = self.head
        expected_prev = "0"
        while cur:
            recomputed = hashlib.sha256(f"{cur.index}|{cur.voter_hash}|{cur.candidate_id}|{cur.timestamp}|{cur.prev_hash}".encode('utf-8')).hexdigest()
            if cur.prev_hash != expected_prev:
                return False, f"Prev-hash mismatch at block {cur.index}", cur.index
            if recomputed != cur.hash:
                return False, f"Hash mismatch at block {cur.index}", cur.index
            expected_prev = cur.hash
            cur = cur.next
        return True, "Chain valid. No tampering detected.", None

    def to_list(self):
        arr = []
        cur = self.head
        while cur:
            arr.append(cur.to_dict())
            cur = cur.next
        return arr

    def load_from_list(self, lst):
        self.head = self.tail = None
        self.size = 0
        for d in lst:
            b = Block.from_dict(d)
            self.append_block(b)

class VoterRegistry:
    def __init__(self):
        self.voters = {}
        self.has_voted = {}

    def register_voter(self, voter_id: str) -> bool:
        if voter_id in self.voters:
            return False
        self.voters[voter_id] = True
        self.has_voted[voter_id] = False
        return True

    def is_registered(self, voter_id: str) -> bool:
        return voter_id in self.voters

    def mark_voted(self, voter_id: str):
        if voter_id in self.has_voted:
            self.has_voted[voter_id] = True

    def already_voted(self, voter_id: str) -> bool:
        return self.has_voted.get(voter_id, False)

    def to_dict(self):
        return {"voters": list(self.voters.keys()), "has_voted": self.has_voted}

    def load_from_dict(self, d):
        self.voters = {vid: True for vid in d.get("voters", [])}
        self.has_voted = d.get("has_voted", {})

class ElectionController:
    def __init__(self, candidates):
        self.ledger = Ledger()
        self.registry = VoterRegistry()
        self.candidates = list(candidates)
        self.election_started = False
        self.next_index = 1

    @staticmethod
    def voter_hash(voter_id: str, salt: str = "SALT123"):
        return hashlib.sha256((voter_id + salt).encode('utf-8')).hexdigest()

    def register_voter(self, voter_id: str) -> bool:
        return self.registry.register_voter(voter_id)

    def start_election(self):
        if self.election_started:
            return False
        self.election_started = True
        return True

    def end_election(self):
        if not self.election_started:
            return False
        self.election_started = False
        return True

    def cast_vote(self, voter_id: str, candidate_choice: int):
        if not self.election_started:
            return False, "Election not started."
        if not self.registry.is_registered(voter_id):
            return False, "Voter not registered."
        if self.registry.already_voted(voter_id):
            return False, "Voter already voted."
        if candidate_choice < 1 or candidate_choice > len(self.candidates):
            return False, "Invalid choice."

        candidate_id = self.candidates[candidate_choice - 1]
        vhash = self.voter_hash(voter_id)
        timestamp = datetime.utcnow().isoformat()
        prev_hash = self.ledger.tail.hash if self.ledger.tail else "0"
        block = Block(self.next_index, vhash, candidate_id, timestamp, prev_hash)
        self.ledger.append_block(block)
        self.next_index += 1
        self.registry.mark_voted(voter_id)
        return True, f"Vote recorded for {candidate_id}."

    def tally_votes(self):
        counts = {c: 0 for c in self.candidates}
        cur = self.ledger.head
        while cur:
            if cur.candidate_id in counts:
                counts[cur.candidate_id] += 1
            cur = cur.next
        return counts

    def verify_ledger(self):
        return self.ledger.verify_chain()

    def tamper_block(self, index: int, new_candidate: str):
        block = self.ledger.get_block_by_index(index)
        if not block:
            return False, "Block not found."
        block.candidate_id = new_candidate
        # don't update hash so tamper is detectable
        return True, f"Block {index} candidate changed to {new_candidate} (tampered)."

    def save_state(self, filename: str = SAVE_FILE):
        data = {
            "candidates": self.candidates,
            "election_started": self.election_started,
            "next_index": self.next_index,
            "ledger": self.ledger.to_list(),
            "registry": self.registry.to_dict()
        }
        with open(filename, "w") as f:
            json.dump(data, f, indent=2)
        return True

    def load_state(self, filename: str = SAVE_FILE):
        if not os.path.exists(filename):
            return False, "No saved state found."
        with open(filename, "r") as f:
            data = json.load(f)
        self.candidates = data.get("candidates", self.candidates)
        self.election_started = data.get("election_started", False)
        self.next_index = data.get("next_index", 1)
        self.ledger.load_from_list(data.get("ledger", []))
        self.registry.load_from_dict(data.get("registry", {}))
        return True, "Loaded state."

# -------------------------
# App state & defaults
# -------------------------
if "ctrl" not in st.session_state:
    # Hardcoded candidates (edit if needed)
    st.session_state.ctrl = ElectionController(["Alice", "Bob", "Charlie"])

# Admin & ledger passwords are SHA256 hashed constants (change raw_password to change)
# Note: storing plain passwords in code is fine for demo; do not do this in prod.
raw_admin_pw = "1234"
raw_ledger_pw = "1234"
ADMIN_PW_HASH = hashlib.sha256(raw_admin_pw.encode()).hexdigest()
LEDGER_PW_HASH = hashlib.sha256(raw_ledger_pw.encode()).hexdigest()

# We will NOT keep persistent 'authenticated' flags (user requested to be prompted every time)
# Instead we use a short-lived 'pending_auth' indicator to present a password field when Admin/Ledger is clicked.
if "pending_auth" not in st.session_state:
    st.session_state.pending_auth = None
if "active_tab" not in st.session_state:
    st.session_state.active_tab = "Home"

# -------------------------
# Helpers
# -------------------------
def hash_pw(pw: str):
    return hashlib.sha256(pw.encode()).hexdigest()

def save_state():
    st.session_state.ctrl.save_state(SAVE_FILE)
    st.success("State saved to " + SAVE_FILE)

def load_state():
    ok, msg = st.session_state.ctrl.load_state(SAVE_FILE)
    if ok:
        st.success(msg)
    else:
        st.warning(msg)

def reset_re_election():
    # reset ledger and voting flags, keep registered voters and candidates
    st.session_state.ctrl.ledger = Ledger()
    st.session_state.ctrl.next_index = 1
    for vid in st.session_state.ctrl.registry.has_voted:
        st.session_state.ctrl.registry.has_voted[vid] = False
    st.session_state.ctrl.start_election()

# -------------------------
# UI layout (kept identical)
# -------------------------
st.title("🎯 Secure E-Voting (Streamlit UI)")

# Top-row tabs evenly spaced using columns
cols = st.columns([1,1,1,1,1])
tab_buttons = []
tab_buttons.append(cols[0].button("Home"))
tab_buttons.append(cols[1].button("Voter"))
# Admin & Ledger require password gating -> we set pending_auth when clicked
tab_buttons.append(cols[2].button("Admin"))
tab_buttons.append(cols[3].button("Ledger"))
tab_buttons.append(cols[4].button("Results"))

# Update active_tab or request auth when a button is pressed
pressed_index = None
for i, pressed in enumerate(tab_buttons):
    if pressed:
        pressed_index = i
        break

if pressed_index is not None:
    label = ["Home","Voter","Admin","Ledger","Results"][pressed_index]
    # If Admin or Ledger pressed -> set pending_auth marker so we can render password field this run
    if label in ("Admin", "Ledger"):
        # Set pending_auth to the label to prompt password (won't persist access after successful unlock)
        st.session_state.pending_auth = label
    else:
        st.session_state.active_tab = label
        st.session_state.pending_auth = None

st.markdown("---")

# If a pending_auth is set, render password prompt (asks each time)
if st.session_state.pending_auth is not None:
    auth_label = st.session_state.pending_auth
    with st.container():
        st.subheader(f"{auth_label} — Password Required")
        pw_key = f"pw_input_{auth_label}"
        pw = st.text_input("Enter password", type="password", key=pw_key)
        submit_key = f"submit_{auth_label}"
        if st.button("Unlock " + auth_label, key=submit_key):
            if pw.strip() == "":
                st.error("Enter a password.")
            else:
                if auth_label == "Admin":
                    if hash_pw(pw) == ADMIN_PW_HASH:
                        st.success("Admin authenticated for this access.")
                        # Open the tab just for this render — do NOT persist authentication
                        st.session_state.active_tab = "Admin"
                        # Clear pending_auth so next time it asks again
                        st.session_state.pending_auth = None
                        # Force a rerun so the UI shows the Admin tab content immediately
                        st.rerun()
                    else:
                        st.error("Wrong admin password.")
                else:  # Ledger
                    if hash_pw(pw) == LEDGER_PW_HASH:
                        st.success("Ledger authenticated for this access.")
                        st.session_state.active_tab = "Ledger"
                        st.session_state.pending_auth = None
                        st.rerun()
                    else:
                        st.error("Wrong ledger password.")

# -------------------------
# HOME
# -------------------------
if st.session_state.active_tab == "Home":
    left, right = st.columns([2,3])
    with left:
        st.header("Overview")
        st.write("This is a Streamlit demo of a blockchain-inspired e-voting system using core data structures.")
        st.write("- Linked list ledger of vote blocks (tamper-evident via SHA-256).")
        st.write("- Voter registry (dict), has_voted tracking, hardcoded candidates.")
        st.write("- Admin controls and tamper demo.")
    with right:
        st.header("Quick Controls")
        if st.button("Load saved state"):
            load_state()
        if st.button("Save current state"):
            save_state()
        st.info("Default admin password (for demo): `adminpass` — ledger password: `ledgerpass`.\nChange in code for security.")

# -------------------------
# VOTER
# -------------------------
elif st.session_state.active_tab == "Voter":
    st.header("Cast your vote")
    ctrl = st.session_state.ctrl
    if not ctrl.election_started:
        st.warning("Election has not started. Wait for admin to start.")
    with st.form("voter_form", clear_on_submit=False):
        vid = st.text_input("Enter Voter ID")
        cols2 = st.columns(len(ctrl.candidates))
        for i,c in enumerate(ctrl.candidates, start=1):
            cols2[i-1].write(f"**{i}. {c}**")
        choice = st.number_input("Candidate number", min_value=1, max_value=len(ctrl.candidates), step=1, value=1)
        submitted = st.form_submit_button("Cast Vote")
        if submitted:
            ok, msg = ctrl.cast_vote(vid.strip(), int(choice))
            if ok:
                st.success(msg)
            else:
                st.error(msg)

# -------------------------
# ADMIN (accessed only when active_tab == "Admin" after correct password)
# -------------------------
elif st.session_state.active_tab == "Admin":
    st.header("Admin Panel")
    ctrl = st.session_state.ctrl
    col1, col2 = st.columns(2)
    with col1:
        st.subheader("Voter Registration")
        with st.form("reg_form", clear_on_submit=True):
            new_vid = st.text_input("Voter ID to register")
            reg_submit = st.form_submit_button("Register Voter")
            if reg_submit:
                if new_vid.strip() == "":
                    st.warning("Enter a valid ID.")
                else:
                    ok = ctrl.register_voter(new_vid.strip())
                    if ok:
                        st.success(f"Voter {new_vid} registered.")
                    else:
                        st.warning("Voter already registered.")
        st.write(f"Registered voters: {len(ctrl.registry.voters)}")

        st.subheader("Election Control")
        if ctrl.election_started:
            if st.button("End Election"):
                ctrl.end_election()
                st.success("Election ended by admin.")
        else:
            if st.button("Start Election"):
                ctrl.start_election()
                st.success("Election started by admin.")
        st.write(f"Election active: {ctrl.election_started}")

        st.subheader("Tamper Demo")
        with st.form("tamper_form"):
            idx = st.number_input("Block index to tamper", min_value=1, step=1, value=1)
            new_c = st.selectbox("Set candidate name (demo)", options=ctrl.candidates)
            tamper_btn = st.form_submit_button("Tamper block")
            if tamper_btn:
                ok, msg = ctrl.tamper_block(int(idx), new_c)
                if ok:
                    st.success(msg)
                else:
                    st.error(msg)

    with col2:
        st.subheader("Ledger & Verification")
        if st.button("Verify ledger integrity"):
            valid, msg, bad = ctrl.verify_ledger()
            if valid:
                st.success(msg)
            else:
                st.error(msg)
                if bad:
                    st.error(f"Tamper suspected at block {bad}")
        if st.button("Save state"):
            save_state()
        if st.button("Load state"):
            load_state()

        st.subheader("Admin quick info")
        st.write("Registered voters (IDs):")
        st.write(list(ctrl.registry.voters.keys()))

# -------------------------
# LEDGER (accessed only when active_tab == "Ledger" after correct password)
# -------------------------
elif st.session_state.active_tab == "Ledger":
    st.header("Ledger Viewer (Read-only)")
    ctrl = st.session_state.ctrl
    lst = ctrl.ledger.to_list()
    if not lst:
        st.info("Ledger is empty.")
    else:
        for d in lst:
            st.markdown(f"**Block {d['index']}** | candidate: {d['candidate_id']}  \n ts: {d['timestamp']}  \n prev: `{d['prev_hash'][:12]}...`  hash: `{d['hash'][:12]}...`")
            st.write("---")

# -------------------------
# RESULTS
# -------------------------
elif st.session_state.active_tab == "Results":
    st.header("Results & Status")
    ctrl = st.session_state.ctrl
    counts = ctrl.tally_votes()
    total_votes = sum(counts.values())
    registered = len(ctrl.registry.voters)

    # Show vote counts
    if total_votes == 0:
        st.info("No votes have been cast yet.")
    else:
        st.subheader("Current vote counts")
        cols = st.columns(3)
        i = 0
        for c, v in counts.items():
            cols[i % 3].metric(label=c, value=v)
            i += 1

    # Determine whether result should be decided
    if total_votes == 0:
        st.write("Voting hasn't started / no votes yet.")
    else:
        # If election active and not all voters voted -> show progress only
        if ctrl.election_started and registered > 0 and total_votes < registered:
            st.info(f"Voting in progress — {total_votes}/{registered} votes cast. Admin can end election to decide early.")
            st.write("No winner/tie declared until election ends or all registered voters vote.")
        else:
            # Decision time (either all voted, or admin ended)
            if registered == 0:
                st.warning("No registered voters — admin should register voters first.")
            else:
                max_votes = max(counts.values()) if counts else 0
                winners = [c for c, v in counts.items() if v == max_votes]
                if len(winners) > 1:
                    st.error("Election tied between: " + ", ".join(winners))
                    st.warning("Automatic re-election will be started (ledger reset & voting flags cleared).")
                    if st.button("Confirm start re-election now"):
                        reset_re_election()
                        st.success("Re-election started. Ledger cleared and election restarted.")
                else:
                    winner = winners[0] if winners else None
                    if winner:
                        st.success(f"Winner: {winner} with {max_votes} votes.")

# -------------------------
# Footer
# -------------------------
st.markdown("---")
st.caption("Streamlit demo — not production-ready. For demo: admin='adminpass', ledger='ledgerpass'. Change in code as needed.")
