from flask import Flask, render_template, request
from eth_account import Account
from web3 import Web3
import json

app = Flask(__name__)
w3 = Web3(Web3.HTTPProvider("https://redcapcoin.pythonanywhere.com/"))  # ← ضع مفتاح Infura الخاص بك

@app.route("/", methods=["GET", "POST"])
def index():
    result = {}
    
    if request.method == "POST":
        action = request.form.get("action")
        
        if action == "create":
            acct = Account.create()
            result["address"] = acct.address
            result["mnemonic"] = acct._mnemonic if hasattr(acct, "_mnemonic") else "لا توجد عبارة افتراضية"
            result["private_key"] = acct.key.hex()
        
        elif action == "import":
            try:
                phrase = request.form.get("mnemonic").strip()
                acct = Account.from_mnemonic(phrase)
                result["address"] = acct.address
                result["mnemonic"] = phrase
                result["private_key"] = acct.key.hex()
            except:
                result["error"] = "عبارة غير صالحة"

        elif action == "send":
            try:
                private_key = request.form.get("private_key").strip()
                to_address = request.form.get("to_address").strip()
                amount = float(request.form.get("amount"))
                
                account = Account.from_key(private_key)
                nonce = w3.eth.get_transaction_count(account.address)
                tx = {
                    "to": to_address,
                    "value": w3.to_wei(amount, "ether"),
                    "gas": 21000,
                    "gasPrice": w3.to_wei("20", "gwei"),
                    "nonce": nonce,
                    "chainId": 1,
                }
                signed_tx = w3.eth.account.sign_transaction(tx, private_key)
                tx_hash = w3.eth.send_raw_transaction(signed_tx.rawTransaction)
                result["tx_hash"] = tx_hash.hex()
            except Exception as e:
                result["error"] = f"فشل الإرسال: {str(e)}"

        elif action == "balance":
            try:
                address = request.form.get("balance_address").strip()
                balance = w3.eth.get_balance(address)
                result["balance_eth"] = w3.from_wei(balance, "ether")
                result["balance_address"] = address
            except Exception as e:
                result["error"] = f"فشل قراءة الرصيد: {str(e)}"

        elif action == "read_contract":
            try:
                contract_address = Web3.to_checksum_address(request.form.get("contract_address"))
                abi = request.form.get("contract_abi")
                function_name = request.form.get("function_name")
                function_args = request.form.get("function_args", "[]")

                args = json.loads(function_args)  # ← باراميترز الدالة من المستخدم
                contract = w3.eth.contract(address=contract_address, abi=abi)
                function = contract.get_function_by_name(function_name)
                result["contract_output"] = function(*args).call()
            except Exception as e:
                result["error"] = f"فشل قراءة العقد: {str(e)}"
    
    return render_template("index.html", result=result)

if __name__ == "__main__":
    app.run(debug=True)
