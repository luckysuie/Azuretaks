# Kubernetes Ingress Controller 
Kubernetes Ingress controlller contains a set of rules which defines signals and direct the correct path to the website.
## 🌐 Imagine You Run a Mall
You have many shops inside:

- Shop A = API service
- Shop B = Web app
- Shop C = Admin dashboard
People come to the main mall gate (one public IP or domain name), but you need a way to direct them to the right shop.

## 🏷️ What an Ingress Does
Ingress is like the signboard at the mall gate that says:
- /api → go to Shop A
- /web → go to Shop B
- /admin → go to Shop C
It’s just the rules telling who goes where.

## 🛡️ What the Ingress Controller Does

The Ingress Controller is like the security guard at the gate.
- It reads the signboard (Ingress rules) and actually guides each visitor to the correct shop
- If someone comes to myapp.com/api, the guard takes them to the API service.
- If someone comes to myapp.com/web, the guard takes them to the Web service.

## 🔒 Bonus Things the Guard Can Do
- Check tickets / HTTPS (SSL) → Makes sure connections are secure.
- Split visitors between multiple shops → Load balancing if one shop gets crowded.
- Redirect visitors → If a shop moves or changes its path.

## ✅ Simple Summary
- Ingress = Signboard (rules of where traffic should go).
- Ingress Controller = Security guard (actually directs the traffic).
- Instead of giving each shop its own separate entrance, you have one main entrance and a smart system to send people to the right place.
