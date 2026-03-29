import { useState, useEffect } from "react";
import { Card, CardContent } from "@/components/ui/card";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { motion } from "framer-motion";

const ADMIN_PASSWORD = "admin123";

function generateTimeSlots() {
  const slots = [];
  let minutes = 10 * 60;
  const end = 24 * 60;
  while (minutes < end) {
    const h = Math.floor(minutes / 60)
      .toString()
      .padStart(2, "0");
    const m = (minutes % 60).toString().padStart(2, "0");
    slots.push(`${h}:${m}`);
    minutes += 20;
  }
  return slots;
}

export default function BookingApp() {
  const today = new Date();
  const [currentMonth, setCurrentMonth] = useState(new Date(2026, 2));
  const [selectedDate, setSelectedDate] = useState(null);
  const [selectedTime, setSelectedTime] = useState(null);
  const [reason, setReason] = useState("");
  const [matricule, setMatricule] = useState("");
  const [appointments, setAppointments] = useState([]);
  const [adminMode, setAdminMode] = useState(false);
  const [blockedDates, setBlockedDates] = useState([]);
  const [passwordInput, setPasswordInput] = useState("");
  const [isAuthenticated, setIsAuthenticated] = useState(false);
  const [searchMatricule, setSearchMatricule] = useState("");

  const timeSlots = generateTimeSlots();

  // 🔥 LOAD DATA
  useEffect(() => {
    const savedAppointments = localStorage.getItem("appointments");
    const savedBlocked = localStorage.getItem("blockedDates");

    if (savedAppointments) setAppointments(JSON.parse(savedAppointments));
    if (savedBlocked) setBlockedDates(JSON.parse(savedBlocked));
  }, []);

  // 💾 SAVE DATA
  useEffect(() => {
    localStorage.setItem("appointments", JSON.stringify(appointments));
  }, [appointments]);

  useEffect(() => {
    localStorage.setItem("blockedDates", JSON.stringify(blockedDates));
  }, [blockedDates]);

  const year = currentMonth.getFullYear();
  const month = currentMonth.getMonth();

  const firstDay = new Date(year, month, 1).getDay();
  const daysInMonth = new Date(year, month + 1, 0).getDate();

  const days = [];
  for (let i = 0; i < firstDay; i++) days.push(null);
  for (let d = 1; d <= daysInMonth; d++) days.push(d);

  const formatDate = (day) => `${year}-${month}-${day}`;

  const isPastDate = (day) => {
    const date = new Date(year, month, day);
    return date < new Date(today.setHours(0, 0, 0, 0));
  };

  const isSunday = (day) => {
    const date = new Date(year, month, day);
    return date.getDay() === 0;
  };

  const isBlocked = (day) => blockedDates.includes(formatDate(day));

  const isBooked = (day, time) => {
    return appointments.some(
      (a) => a.date === formatDate(day) && a.time === time
    );
  };

  const isSameDayBooking = (date, matricule) => {
    return appointments.some(
      (a) => a.date === date && a.matricule === matricule
    );
  };

  const handleBooking = () => {
    const date = formatDate(selectedDate);

    if (!/^\d+$/.test(matricule)) {
      alert("Le matricule doit contenir uniquement des chiffres");
      return;
    }

    if (isSameDayBooking(date, matricule)) {
      alert("Ce matricule a déjà un rendez-vous ce jour-là");
      return;
    }

    if (!reason || !matricule || !selectedDate || !selectedTime) {
      alert("Veuillez remplir tous les champs");
      return;
    }

    setAppointments([
      ...appointments,
      {
        date,
        time: selectedTime,
        reason,
        matricule,
      },
    ]);

    setReason("");
    setMatricule("");
    setSelectedTime(null);
  };

  const deleteAppointment = (index) => {
    const updated = [...appointments];
    updated.splice(index, 1);
    setAppointments(updated);
  };

  const toggleBlockDate = (day) => {
    const date = formatDate(day);
    if (blockedDates.includes(date)) {
      setBlockedDates(blockedDates.filter((d) => d !== date));
    } else {
      setBlockedDates([...blockedDates, date]);
    }
  };

  const handleLogin = () => {
    if (passwordInput === ADMIN_PASSWORD) {
      setIsAuthenticated(true);
      setAdminMode(true);
      setPasswordInput("");
    } else {
      alert("Mot de passe incorrect");
    }
  };

  const handleLogout = () => {
    setIsAuthenticated(false);
    setAdminMode(false);
  };

  const filteredAppointments = appointments.filter((a) =>
    a.matricule.includes(searchMatricule)
  );

  return (
    <div className="p-6 max-w-5xl mx-auto space-y-6">
      <h1 className="text-3xl font-bold text-center capitalize">
        {currentMonth.toLocaleString("fr-FR", {
          month: "long",
          year: "numeric",
        })}
      </h1>

      <div className="flex justify-between items-center">
        <Button onClick={() => setCurrentMonth(new Date(year, month - 1))}>
          ←
        </Button>

        {!isAuthenticated ? (
          <div className="flex gap-2">
            <Input
              type="password"
              placeholder="Mot de passe admin"
              value={passwordInput}
              onChange={(e) => setPasswordInput(e.target.value)}
            />
            <Button onClick={handleLogin}>Connexion</Button>
          </div>
        ) : (
          <Button onClick={handleLogout}>Déconnexion admin</Button>
        )}

        <Button onClick={() => setCurrentMonth(new Date(year, month + 1))}>
          →
        </Button>
      </div>

      <div className="grid grid-cols-7 gap-2">
        {["Dim", "Lun", "Mar", "Mer", "Jeu", "Ven", "Sam"].map(
          (d) => (
            <div key={d} className="text-center font-semibold">
              {d}
            </div>
          )
        )}

        {days.map((day, i) => (
          <div key={i}>
            {day && (
              <motion.div whileHover={{ scale: 1.05 }}>
                <Button
                  className="w-full h-12"
                  variant={
                    selectedDate === day ? "default" : "outline"
                  }
                  disabled={isPastDate(day) || isSunday(day) || isBlocked(day)}
                  onClick={() =>
                    adminMode ? toggleBlockDate(day) : setSelectedDate(day)
                  }
                >
                  {day}
                </Button>
              </motion.div>
            )}
          </div>
        ))}
      </div>

      {!adminMode && selectedDate && (
        <Card className="rounded-2xl shadow-lg">
          <CardContent className="p-4 space-y-4">
            <h2 className="font-semibold">Créneaux disponibles</h2>
            <div className="grid grid-cols-4 gap-2">
              {timeSlots.map((time) => (
                <Button
                  key={time}
                  disabled={isBooked(selectedDate, time)}
                  variant={
                    selectedTime === time ? "default" : "outline"
                  }
                  onClick={() => setSelectedTime(time)}
                >
                  {time}
                </Button>
              ))}
            </div>

            <Input
              placeholder="Motif du rendez-vous"
              value={reason}
              onChange={(e) => setReason(e.target.value)}
            />

            <Input
              placeholder="Matricule (chiffres uniquement)"
              value={matricule}
              onChange={(e) => setMatricule(e.target.value)}
            />

            <Button className="w-full" onClick={handleBooking}>
              Confirmer
            </Button>
          </CardContent>
        </Card>
      )}

      {adminMode && (
        <Card className="rounded-2xl shadow-lg">
          <CardContent className="p-4 space-y-3">
            <h2 className="font-semibold">Gestion des rendez-vous</h2>

            <Input
              placeholder="Rechercher par matricule"
              value={searchMatricule}
              onChange={(e) => setSearchMatricule(e.target.value)}
            />

            {filteredAppointments.map((a, i) => (
              <div
                key={i}
                className="flex justify-between items-center border-b py-2"
              >
                <span>
                  {a.date} - {a.time} | {a.matricule} : {a.reason}
                </span>
                <Button
                  variant="destructive"
                  onClick={() => deleteAppointment(i)}
                >
                  Supprimer
                </Button>
              </div>
            ))}
          </CardContent>
        </Card>
      )}
    </div>
  );
}
