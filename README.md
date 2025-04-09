package com.sus.chatbot;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import org.springframework.http.*;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.client.RestTemplate;
import org.springframework.web.context.annotation.SessionScope;

import javax.persistence.*;
import java.util.*;

@SpringBootApplication
public class ChatbotTriagemApplication {
    public static void main(String[] args) {
        SpringApplication.run(ChatbotTriagemApplication.class, args);
    }

    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}

@Entity
class ChatHistoryEntry {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String role;
    private String message;

    public ChatHistoryEntry() {}
    public ChatHistoryEntry(String role, String message) {
        this.role = role;
        this.message = message;
    }

    public String getRole() { return role; }
    public String getMessage() { return message; }
}

interface ChatHistoryRepository extends org.springframework.data.repository.CrudRepository<ChatHistoryEntry, Long> {
}

@Controller
@RequestMapping("/chat")
@SessionAttributes({"history", "questionIndex"})
class ChatController {
    private static final String[] exampleQuestions = {
        "Qual seu principal sintoma?",
        "Você está com febre?",
        "Tem dificuldade para respirar?",
        "Qual sua idade?"
    };

    @Autowired
    private ChatHistoryRepository historyRepo;

    @Autowired
    private RestTemplate restTemplate;

    @ModelAttribute("history")
    public List<String> history() {
        return new ArrayList<>();
    }

    @ModelAttribute("questionIndex")
    public Integer questionIndex() {
        return 0;
    }

    @GetMapping
    public String chatForm(Model model, @ModelAttribute("history") List<String> history, @ModelAttribute("questionIndex") Integer questionIndex) {
        if (history.isEmpty()) {
            String question = "Bot: " + exampleQuestions[questionIndex];
            history.add(question);
            historyRepo.save(new ChatHistoryEntry("Bot", question));
        }
        model.addAttribute("history", history);
        model.addAttribute("input", new UserInput());
        return "chat";
    }

    @PostMapping
    public String chatSubmit(@ModelAttribute UserInput input,
                             @ModelAttribute("history") List<String> history,
                             @ModelAttribute("questionIndex") Integer questionIndex,
                             Model model) {
        String userText = input.getText();
        history.add("Usuário: " + userText);
        historyRepo.save(new ChatHistoryEntry("Usuário", userText));

        BotResponse result = getBotResponse(questionIndex, userText);
        history.add("Bot: " + result.response);
        historyRepo.save(new ChatHistoryEntry("Bot", result.response));

        if (result.nextIndex != -1) {
            String question = "Bot: " + exampleQuestions[result.nextIndex];
            history.add(question);
            historyRepo.save(new ChatHistoryEntry("Bot", question));
        }

        model.addAttribute("history", history);
        model.addAttribute("questionIndex", result.nextIndex);
        model.addAttribute("input", new UserInput());
        return "chat";
    }

    @PostMapping("/restart")
    public String restartChat(SessionStatus status) {
        status.setComplete();
        return "redirect:/chat";
    }

    private BotResponse getBotResponse(int index, String answer) {
        String lower = answer.toLowerCase();
        String response = "";
        int next = index + 1;

        if (index == 0) {
            response = classifySymptomWithAI(answer);
        } else if (index == 2 && (lower.contains("sim") || lower.contains("muito"))) {
            response = "Dificuldade para respirar pode indicar gravidade.";
        } else if (index == 3) {
            try {
                int idade = Integer.parseInt(answer);
                if (idade >= 60) {
                    response = "Idosos têm maior risco. Atenção redobrada.";
                } else {
                    response = "Obrigado por informar sua idade.";
                }
            } catch (NumberFormatException e) {
                response = "Não entendi a idade. Tudo bem, vamos continuar.";
            }
        } else {
            response = "Obrigado pela informação.";
        }

        if (next >= exampleQuestions.length) {
            response += "\n\n✅ Recomendação: Dirija-se à UPA mais próxima se os sintomas persistirem ou piorarem.";
            next = -1;
        }

        return new BotResponse(response, next);
    }

    private String classifySymptomWithAI(String symptom) {
        try {
            HttpHeaders headers = new HttpHeaders();
            headers.setContentType(MediaType.APPLICATION_JSON);
            headers.set("Authorization", "Bearer SUA_API_KEY_AQUI");

            Map<String, Object> body = new HashMap<>();
            body.put("model", "gpt-3.5-turbo");
            body.put("messages", List.of(
                Map.of("role", "system", "content", "Você é um especialista médico."),
                Map.of("role", "user", "content", "Classifique esse sintoma: " + symptom)
            ));

            HttpEntity<Map<String, Object>> request = new HttpEntity<>(body, headers);
            ResponseEntity<Map> response = restTemplate.postForEntity("https://api.openai.com/v1/chat/completions", request, Map.class);

            List<Map<String, Object>> choices = (List<Map<String, Object>>) response.getBody().get("choices");
            if (choices != null && !choices.isEmpty()) {
                Map<String, Object> message = (Map<String, Object>) choices.get(0).get("message");
                return message.get("content").toString();
            }
        } catch (Exception e) {
            return "Erro ao consultar IA: " + e.getMessage();
        }

        return "Não foi possível classificar o sintoma no momento.";
    }

    static class BotResponse {
        String response;
        int nextIndex;

        BotResponse(String response, int nextIndex) {
            this.response = response;
            this.nextIndex = nextIndex;
        }
    }

    static class UserInput {
        private String text;

        public String getText() {
            return text;
        }

        public void setText(String text) {
            this.text = text;
        }
    }
}
