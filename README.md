# projecttest
Demo : https://thriving-torte-197d45.netlify.app/

import org.springframework.data.jpa.domain.Specification;

import javax.persistence.criteria.*;
import java.time.LocalDate;

public class TaskViewSpecifications {

    public static Specification<TaskView> calculateAverageElapsedTimeWithMinMax(Long specificStatusId) {
        return (Root<TaskView> root, CriteriaQuery<?> query, CriteriaBuilder cb) -> {

            // COALESCE(exitdate, CURRENT_DATE)
            Expression<LocalDate> exitDate = cb.coalesce(root.get("exitDate"), cb.literal(LocalDate.now()));

            // MIN(entryDate) and MAX(exitDate)
            Expression<LocalDate> minEntryDate = cb.least(root.get("entryDate"));
            Expression<LocalDate> maxExitDate = cb.greatest(exitDate);

            // CASE to handle specific status and create 'Fake Stage'
            Expression<String> stageId = cb.selectCase()
                    .when(cb.equal(root.get("status"), specificStatusId), cb.literal("Fake Stage"))
                    .otherwise(root.get("stageId"));

            // Calculate elapsed time: (maxExitDate - minEntryDate) in hours
            Expression<Long> elapsedTimeInSeconds = cb.function(
                    "EXTRACT", Long.class,
                    cb.literal("EPOCH"),
                    cb.diff(maxExitDate, minEntryDate)
            );
            Expression<Double> elapsedTimeInHours = cb.quot(elapsedTimeInSeconds, 3600.0);

            // Group by stage_id
            query.groupBy(stageId);

            // Select stage_id and average elapsed time
            query.multiselect(
                    stageId, // Stage ID or Fake Stage
                    cb.avg(elapsedTimeInHours) // Average elapsed time in hours
            );

            query.orderBy(cb.asc(stageId)); // Order results by stage ID

            return null; // No specific filtering condition
        };
    }
}
