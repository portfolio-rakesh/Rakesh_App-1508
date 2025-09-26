#include <iostream>
#include <vector>
#include <string>
#include <unordered_map>
#include <sstream>
#include <algorithm>
#include <iterator>  // FIX for istream_iterator

enum class QuestionType { SINGLE_CHOICE, MULTIPLE_CHOICE, TEXT };

struct Option {
    int id;
    std::string value;
    bool is_correct;
};

struct Question {
    int id;
    std::string text;
    QuestionType type;
    std::vector<Option> options;
};

struct Quiz {
    int id;
    std::string title;
    std::vector<Question> questions;
};

class QuizService {
    std::unordered_map<int, Quiz> quizzes_;
    int quiz_id_ = 1;
    int ques_id_ = 1;
    int opt_id_ = 1;

    size_t countWords(const std::string& s) {
        std::istringstream iss(s);
        return std::distance(std::istream_iterator<std::string>(iss), std::istream_iterator<std::string>());
    }

public:
    int createQuiz(const std::string& title) {
        Quiz quiz{quiz_id_++, title, {}};
        quizzes_[quiz.id] = quiz;
        return quiz.id;
    }

    bool validateQuestion(const std::string& text, QuestionType type, const std::vector<std::string>& options, int correct) {
        if (type == QuestionType::TEXT && text.length() > 300) return false;
        if ((type == QuestionType::SINGLE_CHOICE || type == QuestionType::MULTIPLE_CHOICE) && (options.size() < 2)) return false;
        if (type == QuestionType::SINGLE_CHOICE && (correct < 0 || correct >= int(options.size()))) return false;
        return true;
    }

    int addQuestion(int quiz_id, const std::string& text, QuestionType type, const std::vector<std::string>& options, int correct) {
        if (!quizzes_.count(quiz_id)) return -1;
        if (!validateQuestion(text, type, options, correct)) return -1;
        Question q{ques_id_++, text, type};
        for (size_t i = 0; i < options.size(); ++i) {
            q.options.push_back({opt_id_++, options[i], (type == QuestionType::SINGLE_CHOICE) && (int(i) == correct)});
        }
        quizzes_[quiz_id].questions.push_back(q);
        return q.id;
    }

    std::vector<Question> getQuestions(int quiz_id) {
        if (!quizzes_.count(quiz_id)) return {};
        auto questions = quizzes_[quiz_id].questions;
        // Hide correct answers
        for (auto& q : questions) for (auto& o : q.options) o.is_correct = false;
        return questions;
    }

    int submitAnswers(int quiz_id, const std::unordered_map<int, int>& answers) {
        if (!quizzes_.count(quiz_id)) return -1;
        int score = 0;
        for (const auto& q : quizzes_[quiz_id].questions) {
            auto it = answers.find(q.id);
            if (it != answers.end()) {
                for (const auto& o : q.options)
                    if (o.id == it->second && o.is_correct) score++;
            }
        }
        return score;
    }

    std::vector<Quiz> listQuizzes() {
        std::vector<Quiz> result;
        for (const auto& p : quizzes_) result.push_back(p.second);
        return result;
    }
};

int main() {
    QuizService api;
    int quizId = api.createQuiz("C++ Fundamentals");
    api.addQuestion(quizId, "Largest integer type in C++?", QuestionType::SINGLE_CHOICE, {"int", "long long", "short"}, 1);
    api.addQuestion(quizId, "Result of 2+2?", QuestionType::SINGLE_CHOICE, {"2", "3", "4"}, 2);

    auto questions = api.getQuestions(quizId);
    std::cout << "Quiz: " << quizId << "\n";
    for (auto& q : questions) {
        std::cout << q.text << "\n";
        for (auto& o : q.options) std::cout << "  " << o.value << "\n";
    }

    std::unordered_map<int, int> answers = {
        {questions[0].id, questions[0].options[1].id}, // correct: long long
        {questions[1].id, questions[1].options[2].id}  // correct: 4
    };
    int userScore = api.submitAnswers(quizId, answers);
    std::cout << "Your score: " << userScore << "\n";
    return 0;
}
