# vdsggbd
import { useState } from "react";
import { Card, CardContent } from "@/components/ui/card";
import { Input } from "@/components/ui/input";
import { Button } from "@/components/ui/button";
import { motion } from "framer-motion";

const exampleQuestions = [
  "Qual seu principal sintoma?",
  "Você está com febre?",
  "Tem dificuldade para respirar?",
  "Qual sua idade?",
];

const getBotResponse = (questionIndex, answer, history) => {
  const lower = answer.toLowerCase();
  let response = "";
  let next = questionIndex + 1;

  if (questionIndex === 0) {
    if (lower.includes("dor no peito") || lower.includes("falta de ar")) {
      response = "Entendi. Esses sintomas podem ser graves.";
    } else {
      response = "Obrigado pela informação.";
    }
  } else if (questionIndex === 2 && (lower.includes("sim") || lower.includes("muito"))) {
    response = "Dificuldade para respirar pode indicar gravidade.";
  } else if (questionIndex === 3) {
    const idade = parseInt(answer);
    if (!isNaN(idade) && idade >= 60) {
      response = "Idosos têm maior risco. Atenção redobrada.";
    } else {
      response = "Obrigado por informar sua idade.";
    }
  }

  if (next >= exampleQuestions.length) {
    response +=
      "\n\n✅ Recomendação: Dirija-se à UPA mais próxima se os sintomas persistirem ou piorarem.";
    next = -1;
  }

  return { response, next };
};

export default function ChatbotTriagem() {
  const [questionIndex, setQuestionIndex] = useState(0);
  const [input, setInput] = useState("");
  const [history, setHistory] = useState([
    { sender: "bot", text: exampleQuestions[0] },
  ]);

  const handleSend = () => {
    if (!input.trim()) return;

    const userMessage = { sender: "user", text: input };
    const { response, next } = getBotResponse(questionIndex, input, history);
    const botMessage = { sender: "bot", text: response };

    setHistory([...history, userMessage, botMessage]);
    setInput("");
    if (next !== -1) {
      setTimeout(() => {
        setHistory((prev) => [...prev, { sender: "bot", text: exampleQuestions[next] }]);
        setQuestionIndex(next);
      }, 600);
    }
  };

  return (
    <div className="max-w-xl mx-auto p-4">
      <Card className="h-[70vh] overflow-y-auto p-4 space-y-2 bg-gray-50">
        <CardContent>
          {history.map((msg, idx) => (
            <motion.div
              key={idx}
              initial={{ opacity: 0, y: 5 }}
              animate={{ opacity: 1, y: 0 }}
              className={`p-2 rounded-xl max-w-sm ${
                msg.sender === "bot" ? "bg-white text-left" : "bg-blue-100 text-right ml-auto"
              }`}
            >
              {msg.text}
            </motion.div>
          ))}
        </CardContent>
      </Card>
      <div className="flex gap-2 mt-4">
        <Input
          placeholder="Digite sua resposta..."
          value={input}
          onChange={(e) => setInput(e.target.value)}
          onKeyDown={(e) => e.key === "Enter" && handleSend()}
        />
        <Button onClick={handleSend}>Enviar</Button>
      </div>
    </div>
  );
}
